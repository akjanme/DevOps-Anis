# Docker Learning — File 4 of 7 — Networking, Storage, Compose, Registries & Troubleshooting

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 6 (Networking) · Part 7 (Storage) · Part 8 (Compose) · Part 9 (Registries) · Part 10 (Troubleshooting playbook)
>
> ⬅ Previous: File 3   |   ➡ Next: File 5

---

# Part 6 — Networking

## The four modes

```bash
docker network ls
```

| Mode | What it means | When to use |
|---|---|---|
| **bridge** (default) | Private network on this host; you must publish ports to reach in | The default. Fine, but see below |
| **user-defined bridge** | Same, plus containers can find each other **by name** | 🏭 **What you should actually use** |
| **host** | No network isolation — the container shares the host's network directly | Maximum speed, no port mapping, weaker isolation. ⚙️ Native on Linux; on Mac/Windows Desktop it is an opt-in setting |
| **none** | No network at all | Batch jobs, security sandboxes |

## How the default bridge works

```
  YOUR SERVER
  +-------------------------------------------------------------+
  |            docker0  (a virtual network switch)               |
  |               |                      |                       |
  |          virtual cable          virtual cable                |
  |               |                      |                       |
  |   +-----------+--------+   +---------+----------+            |
  |   | Container A        |   | Container B        |            |
  |   | eth0: 172.17.0.2   |   | eth0: 172.17.0.3   |            |
  |   +--------------------+   +--------------------+            |
  +-------------------------------------------------------------+
         |
    the server's real network card (e.g. 10.0.0.5)

  "-p 8080:80" adds a firewall rule:
     traffic arriving at server:8080  -->  forwarded to 172.17.0.2:80
```

Two things worth understanding:

- Containers can reach the internet (their traffic is translated to look like it came from the server), but the internet cannot reach them unless you publish a port.
- `-p 8080:80` is not magic — it writes a Linux firewall (iptables) rule. You can see them: `sudo iptables -t nat -L DOCKER`.

🔴 **Mistake / 🏭 Serious:** `-p 5432:5432` listens on **all** network interfaces, and Docker's firewall rules can bypass a host firewall like `ufw`. Teams have accidentally exposed databases to the entire internet this way. Bind explicitly when a port should stay local:

```bash
docker run -p 127.0.0.1:5432:5432 postgres:16     # only reachable from this machine
```

## Containers finding each other by name

On the **default** bridge network, containers can only reach each other by IP address — and IP addresses change on every restart. That is useless.

On a **network you create yourself**, Docker runs a tiny DNS server inside each container, and **container names become hostnames**:

```bash
docker network create appnet

docker run -d --name db  --network appnet postgres:16
docker run -d --name api --network appnet myapp

# inside "api", the database is simply reachable at:  db:5432
```

🏭 **Always create a network for your app.** Never hardcode container IP addresses.

## The connection rules people get wrong

This little table saves hours:

| I am... | I want to reach... | I use... |
|---|---|---|
| A container | Another container on the same network | **Its name and its internal port**: `db:5432` |
| My laptop / the outside | A container | `localhost:<published port>`: `localhost:8080` |
| A container | The host machine itself | `host.docker.internal` (on Linux, add `--add-host=host.docker.internal:host-gateway`) |

🔴 **Mistake:** using the *published* port for container-to-container traffic. If you ran `-p 8080:80`, another container still connects to port **80**, not 8080. The published port only exists on the host side.

## Network debugging tools

```bash
docker network inspect appnet             # who is on this network
docker exec api getent hosts db           # does the name resolve?
docker exec api curl -v http://db:5432    # can it connect?

# The professional trick: a container full of network tools, sharing
# another container's network — works even if your app image has no shell
docker run --rm -it --network container:api nicolaka/netshoot ss -tlnp
```

## Advanced networking (know these exist)

- **macvlan / ipvlan** — gives a container its own MAC address and IP on your real office/home network, as if it were another physical machine. Useful for legacy appliances and home labs. Two gotchas: the host itself usually *cannot* reach its own macvlan containers, and cloud providers (AWS/Azure) generally block it — use `ipvlan` there.
- **MTU mismatch** — ⚙️ 🎯 A great scenario question. Symptom: small requests work, large uploads or HTTPS handshakes hang forever. Cause: your host is on a VPN or cloud overlay with a smaller maximum packet size (e.g. 1450), but Docker assumes 1500, so big packets vanish silently. Fix: set `"mtu": 1450` in Docker's config (Part 13).
- **Isolation between containers** — containers on the same network can talk to each other freely. Separating apps onto **different networks** is the way to firewall them from each other.
- **`--network container:X`** — makes one container share another's network completely. This is exactly how Kubernetes pods work internally: containers in a pod share one network and talk over `localhost`.

---

# Part 7 — Storage: keeping data alive

## The problem

The container's writable layer disappears when the container is deleted. So anything that must survive — database files, uploaded images, generated reports — has to live outside it.

## Three ways to store data

```
  +---------------------- container --------------------------+
  |  /var/lib/postgresql/data     /app/src          /tmp       |
  +----------|--------------------|-----------------|---------+
             |                    |                 |
         VOLUME              BIND MOUNT           TMPFS
   Docker manages it     a folder you pick     lives in memory
   (best for data)       on the host           only, never on disk
                         (best for dev)        (secrets, scratch)
```

```bash
# VOLUME — the production choice for real data
docker volume create pgdata
docker run -d -v pgdata:/var/lib/postgresql/data postgres:16

# BIND MOUNT — map a real folder in; great for development and config files
docker run -d -v $(pwd)/src:/app/src myapp
docker run -d -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx   # :ro = read-only

# TMPFS — memory only, disappears completely on stop (Linux)
docker run -d --tmpfs /tmp:rw,size=64m myapp
```

| | Volume | Bind mount | tmpfs |
|---|---|---|---|
| Where it lives | Docker's own storage area | Any folder you choose | RAM |
| Survives container deletion | **Yes** | Yes | No |
| Works on any host | Yes | No — the path must exist there | Yes |
| Can use network storage (NFS, cloud) | Yes | No | No |
| Best for | Databases, app data | Dev code, config files | Secrets, temp files |

## Behaviors that surprise people

**1. Empty volumes get filled from the image; bind mounts do not.**

Mount a *new empty named volume* onto a folder, and Docker copies the image's existing files there first. Mount an *empty host folder* on the same path, and it simply covers up whatever the image had.

🔴 That is why mounting an empty folder over `/usr/share/nginx/html` makes your website disappear. 🎯 Common trap question.

**2. Deleting a container does not delete its volumes.**

Orphaned volumes pile up for months. `docker volume prune` on an old server often frees tens of gigabytes. But be careful — that same carelessness in reverse (`docker system prune --volumes`) has deleted live databases.

**3. File permissions.**

Volumes keep the host's file ownership. If your container runs as user 1000 but the folder belongs to root, your app gets "permission denied" and crash-loops.

⚙️ On Mac/Windows, Docker Desktop smooths this over — which is exactly why "it worked on my Mac but permission-denied on the Linux server" happens so often.

**4. Can you run a database in Docker?**

🎯 A favorite interview question, and the naive answers ("never!") are wrong. The real answer: **yes, if** the data is on a volume, memory limits are set, and the stop timeout is long enough for the database to shut down cleanly (`--stop-timeout 60`). At large scale many teams still prefer a managed database — but that is an operational choice, not a Docker limitation.

---

# Part 8 — Docker Compose: running several containers together

Real apps are not one container. They are an app plus a database plus a cache. Compose lets you describe the whole thing in one file.

⚙️ Use `docker compose` (two words — the modern plugin). The old `docker-compose` (with a hyphen) is retired.

## A complete example

```yaml
# compose.yaml
services:
  api:
    build: .
    ports:
      - "8080:8000"
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/app   # "db" = the service name
    depends_on:
      db:
        condition: service_healthy     # wait until the DB is really ready
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10

volumes:
  pgdata:
```

```bash
docker compose up -d          # build/create/start everything
docker compose ps             # status, including health
docker compose logs -f api    # follow one service's logs
docker compose exec api sh    # shell into a service
docker compose up -d --build  # rebuild after code changes
docker compose down           # stop and remove containers + network (VOLUMES SURVIVE)
docker compose down -v        # ...and delete the volumes too (destroys your data)
```

## Key points

- Compose automatically creates a private network, so **services reach each other by service name** (`db`, `api`). No IP addresses.
- 🔴 `depends_on` on its own only controls **start order**, not readiness. A database container can be "started" but still be initializing. Use `condition: service_healthy` with a healthcheck, as above. This is a classic source of flaky startup.
- 🎯 **Trap:** `docker compose down` does **not** delete your volumes. Only `down -v` does. Interviewers ask this in both directions.

## Compose features worth knowing

**Multiple files for different environments:**

```bash
# compose.yaml           — the shared base
# compose.override.yaml  — development extras (merged automatically)
# compose.prod.yaml      — production settings
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose config     # print the final merged result — the way to debug confusion
```

**Optional services** (heavy tools you do not always want):

```yaml
  pgadmin:
    image: dpage/pgadmin4
    profiles: ["debug"]        # only starts with: docker compose --profile debug up
```

**Live development mode** (Compose 2.22+) — replaces the old bind-mount-plus-file-watcher hacks:

```yaml
    develop:
      watch:
        - action: sync          # copy changed source files straight in
          path: ./src
          target: /app/src
        - action: rebuild       # changing this file rebuilds the image
          path: requirements.txt
```

Run it with `docker compose watch`.

**Where Compose stops:** it runs everything on **one machine**. No automatic failover, no spreading across servers. That limit is exactly where Kubernetes begins (Part 15).

---

# Part 9 — Registries: storing and sharing images

A **registry** is a server that stores images. Docker Hub is the public one; companies use private ones (AWS ECR, Google Artifact Registry, GitHub GHCR, Harbor).

```
  your machine / CI                registry                servers
  docker build   ------------>   docker push   ------->   docker pull
                                                          docker run
```

```bash
docker login ghcr.io
docker build -t ghcr.io/myteam/myapp:1.4.2 .
docker push ghcr.io/myteam/myapp:1.4.2
```

Pushes and pulls only transfer layers the other side does not already have (Part 3), which is why the second push of a similar image is fast.

## Tagging: the discipline that makes production predictable

🔴 **The `latest` problem.** `latest` is not "the newest version." It is just a default label. It can point to something old. It can change under you. Two servers pulling `latest` an hour apart can end up running different code.

🏭 **What good teams do:**

- CI builds an image once and tags it with something **unique and permanent** — the git commit ID (`myapp:git-abc1234`) and/or a version (`myapp:1.4.2`).
- Production deploys by **digest** (the exact fingerprint) or by those never-reused tags.
- `latest` is never what production runs.

Why it matters: you must be able to answer "what exactly is running right now?" and "put back what ran yesterday" with certainty. Mutable tags make both impossible.

🎯 **Interview:** "What is your tagging strategy?" Answer: *"Immutable tags from CI — git SHA plus semantic version — and production pins the digest. `latest` is for convenience locally, never for deployment."*

## Images for different CPU types

Modern laptops are ARM. Many servers are still Intel/AMD (and many new ones are ARM too, because they are cheaper).

One tag like `nginx:1.27` actually points to a small list of images, one per CPU type. Your machine automatically pulls the right one.

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myteam/myapp:1.4.2 --push .
docker buildx imagetools inspect myteam/myapp:1.4.2    # see what is inside the tag
```

🔴 **Mistake:** building on an Apple Silicon Mac and deploying to an Intel server. The container fails instantly with an "exec format error." Build for both platforms, or build for the target explicitly.

## Practical registry advice 🏭

- Use a **private registry** or a caching mirror for production, so you do not depend on Docker Hub being up (and do not hit its download limits).
- Turn on **automatic scanning** in the registry (Part 11).
- Set a **cleanup policy** — old CI images will otherwise fill terabytes.

---

# Part 10 — When things break: the troubleshooting playbook

This is the part that gets you hired and keeps you calm at 3 AM. Learn the loop, learn the exit codes, learn the six scenarios.

## The debugging loop

```
  1. docker ps -a       ->  Is it running? If not, what EXIT CODE?
  2. docker logs <name> ->  What did the application say before dying?
  3. docker inspect     ->  Was it killed for using too much memory?
                            How many times has it restarted?
                            Are the settings/env/mounts what I expect?
  4. docker exec        ->  Get inside and look around
  5. docker stats       ->  Is it starving for memory or CPU?
```

Work in that order. Most problems are solved by step 2.

## Exit codes — memorize this table

When a container stops, it leaves a number. The number tells you what happened.

| Code | Means | Usual cause |
|---|---|---|
| **0** | Finished normally | A batch job that completed. **Or** — a web server that "finished" because it started itself in the background and process 1 exited. Run servers in foreground mode (e.g. `nginx -g "daemon off;"`) |
| **1** | Application error | An unhandled exception. Read `docker logs` |
| **126** | Cannot execute the file | Missing execute permission, **or** wrong CPU type (ARM image on an Intel server) |
| **127** | Command not found | Typo in `CMD`, or the program is not in the image (e.g. `bash` does not exist in Alpine — use `sh`) |
| **137** | Killed hard (128+9) | 🔴 **Usually out of memory.** Or `docker stop` gave up waiting and killed it |
| **139** | Crashed (128+11, segfault) | A native crash. On Alpine, often a musl-vs-glibc mismatch |
| **143** | Politely terminated (128+15) | Normal during a deploy — this is what a clean `docker stop` looks like |

## Scenario 1: The container keeps restarting

```bash
docker ps -a
# STATUS: Restarting (137) 8 seconds ago

docker logs --tail 200 web
docker inspect -f '{{.State.OOMKilled}} {{.RestartCount}}' web
```

- **137 and `OOMKilled=true`** → it used more memory than its limit. Either raise `--memory` or fix the leak. Look at `docker stats` over time: steadily climbing memory means a leak, a sudden spike means one heavy request.
- **137 and `OOMKilled=false`** → something else killed it, or it ignored the shutdown signal until the timeout ran out (Part 4's process-1 problem).
- **Exit 1** → your application crashed. The stack trace is in the logs.
- **Exit 0 immediately** → your program started itself in the background. Run it in the foreground.

## Scenario 2: "It works on my machine but not on the server"

Check these in order — this list is ranked by how often each is the answer:

1. **Different environment variables or config.** `docker inspect -f '{{.Config.Env}}' web`
2. **The app listens on `127.0.0.1` inside the container.** 🔴 This is a top-three classic. Inside a container, `localhost` means *only inside that container*. Your app must listen on `0.0.0.0` to accept connections from outside.
3. **Wrong CPU architecture** (Mac ARM build → Intel server). Exit code 126 or "exec format error."
4. **A bind mount that exists on your laptop but not on the server.**
5. **Filename case.** macOS ignores case; Linux does not. `Config.py` vs `config.py` breaks only on the server.
6. **Alpine's musl library** breaking a compiled dependency.

## Scenario 3: The server ran out of disk

```bash
df -h /var/lib/docker
docker system df -v                                        # what is using the space
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h    # log files
```

Usual culprits, in order:

1. **Log files that were never rotated.** 🔴 The single most common preventable Docker incident. Permanent fix in Part 13.
2. **Old images** from repeated deploys → `docker image prune -a`
3. **Forgotten volumes** → `docker volume prune` (carefully — check what they are first)
4. **Build cache** on CI machines → `docker builder prune`

🏭 Production servers should do this on a schedule, not in an emergency.

## Scenario 4: Container A cannot reach container B

```bash
docker network inspect appnet         # are they actually on the SAME network?
docker exec a getent hosts b          # does the name resolve?
docker exec a curl -v http://b:5432   # remember: INTERNAL port, not the published one
```

Causes, in order: they are on different networks; you used the published port instead of the internal one; you are on the default bridge where names do not resolve; B is listening on `127.0.0.1`; or B simply is not ready yet (use healthchecks, Part 8).

## Scenario 5: The image has no shell

Small images (distroless, scratch) have no `sh`, no `bash`, no tools. `docker exec -it app sh` fails.

You do not need a shell inside the image — you can **borrow another container's view of the world**:

```bash
# Join the target's process list and network with a container full of tools
docker run --rm -it --pid container:app --network container:app nicolaka/netshoot
# Now ps, ss, curl, dig all work AS IF you were inside the app container

# Or, from the host directly:
sudo nsenter -t $(docker inspect -f '{{.State.Pid}}' app) -n ss -tlnp
```

🎯 This is the senior answer to "how do you debug a container with no shell." (Docker Desktop also has `docker debug` for this.)

## Scenario 6: Docker itself is unhappy

```bash
systemctl status docker           # is the service running?
journalctl -u docker --since '1 hour ago'    # daemon logs
docker events                     # live stream of what Docker is doing
```

- `docker` commands hang → check the daemon and the socket permissions.
- A storm of events → some container is stuck in a restart loop, eating the machine.
- High load but idle containers → often a container whose process 1 never cleans up finished child processes. Fix with `--init` (Part 4).

---

