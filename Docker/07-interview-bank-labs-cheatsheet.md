# Docker Learning — File 7 of 7 — Interview Question Bank, Labs & Cheat Sheet

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 16 (70 interview questions) · Part 17 (9 hands-on labs) · final self-check · command cheat sheet
>
> ⬅ Previous: File 6   |   ➡ Next: —

---

# Part 16 — 70 interview questions with answers

**How to use this:** cover the answer, say yours out loud, then compare. Speaking is the skill being tested, not recognizing. The part number tells you where to re-read if an answer felt shaky.

## Beginner (1–15)

**1. What is Docker and what problem does it solve?**
Docker packages an application together with everything it needs to run into one image, so the same package runs identically on a laptop, in CI, and in production. It removes environment drift — the biggest historical cause of failed deployments. *(Part 0)*

**2. Image vs container?**
An image is the immutable, built package — like a class. A container is a running instance of it with its own writable layer — like an object. One image, many containers. *(Part 1)*

**3. Container vs virtual machine?**
A VM simulates hardware and runs its own full operating system with its own kernel. A container is an isolated process sharing the host kernel. Containers start in milliseconds and are far lighter, but have a weaker isolation boundary. *(Part 15)*

**4. What is a Dockerfile?**
A text file of build steps that produces an image. Most steps create a layer. *(Part 5)*

**5. What does `docker run -d -p 8080:80 nginx` do?**
Starts an nginx container in the background and forwards port 8080 on the host to port 80 inside the container. *(Part 1)*

**6. `docker stop` vs `docker kill`?**
`stop` politely asks the app to shut down, waits 10 seconds, then kills it. `kill` kills it immediately. Graceful shutdown only works if the app listens for the signal *and* you used the bracket form of CMD. *(Part 4)*

**7. How do you see what a container printed?**
`docker logs <name>`, with `-f` to follow. It shows only what process 1 printed to the screen. *(Part 4)*

**8. How do you get a shell inside a running container?**
`docker exec -it <name> sh` (or `bash` if the image has it). *(Part 4)*

**9. `COPY` vs `ADD`?**
`COPY` copies files. `ADD` also unpacks tar archives and can fetch URLs. Use `COPY` unless you specifically need tar extraction. *(Part 5)*

**10. What is Docker Hub?**
The public registry — a server that stores images. Companies typically use private ones. *(Part 9)*

**11. How do you list running and stopped containers?**
`docker ps` for running, `docker ps -a` for all. Stopped containers still occupy disk until removed. *(Part 4)*

**12. What does `-it` mean?**
Interactive plus a terminal — needed when you want to type into the container. *(Part 4)*

**13. How do two containers on one host talk to each other?**
Put them on a network you created, then use container names as hostnames with the internal port. Never hardcode IPs. *(Part 6)*

**14. How do you keep data after a container is deleted?**
Store it in a volume (or bind mount). The container's own writable layer is deleted with the container. *(Part 7)*

**15. What is Docker Compose for?**
Describing and running a multi-container application on one machine from a single YAML file. *(Part 8)*

## Intermediate (16–35)

**16. Walk me through exactly what happens when you run `docker run nginx`.**
The CLI sends a request to the Docker daemon over a socket. The daemon has containerd check for the image locally and download the layers if missing, verifying each by fingerprint. The storage driver stacks the read-only layers and adds a writable layer. Docker creates a virtual network cable into the bridge and assigns an IP. containerd writes a runtime spec and starts a shim, which calls runc. runc creates the namespaces and cgroups, applies security filters, switches to the new root filesystem, and starts your program as process 1. runc then exits; the shim remains as the parent. *(Part 2 — narrating this well is a major differentiator)*

**17. What are namespaces and cgroups?**
Namespaces control what a process can *see* (its own process list, network, filesystem, hostname). Cgroups control how much it can *use* (memory, CPU, process count). Together with a layered filesystem, they are the whole of containerization. *(Part 2)*

**18. Why is the Docker socket dangerous?**
Anyone who can write to it can ask the daemon to start a privileged container mounting the whole host filesystem. Socket access is effectively root on the machine — which is also why the `docker` group is not casual. *(Part 2)*

**19. What is the shim for?**
It is the container's parent process, so the Docker daemon can be restarted or upgraded without killing running containers. That is what `live-restore` relies on. *(Parts 2, 13)*

**20. Explain layer caching and why Dockerfile order matters.**
Each step's result is cached and reused unless that step or its input files change — and once a layer is rebuilt, all layers after it rebuild too. So put rarely-changing steps first and copy source code last, otherwise every code change reinstalls all dependencies. *(Part 5)*

**21. ENTRYPOINT vs CMD?**
ENTRYPOINT is the program that always runs; CMD provides default arguments. Arguments you type after the image name replace CMD, not ENTRYPOINT. *(Part 5)*

**22. Exec form vs shell form — why does it matter?**
Bracket form makes your app process 1, so it receives the shutdown signal. Shell form puts a shell at process 1, which swallows the signal — so every `docker stop` becomes a hard kill after 10 seconds and deploys drop requests. *(Parts 4, 5)*

**23. What is a multi-stage build and why use it?**
Build in one stage with all the compilers and tools, then copy only the finished artifact into a clean minimal stage. Result: much smaller images with far less that can be vulnerable. *(Part 5)*

**24. My image is 1.2 GB. How do you shrink it?**
Smaller base (`-slim` or distroless), multi-stage build, correct layer ordering, a `.dockerignore`, and combining related `RUN` steps. Quote before/after sizes. *(Part 5)*

**25. Volume vs bind mount?**
A volume is managed by Docker, portable between hosts, and supports storage drivers — the right choice for real data. A bind mount maps a specific host folder in — great for development and config, but tied to that machine's layout. *(Part 7)*

**26. What does EXPOSE do?**
Documentation only. It publishes nothing. Only `-p` (or `-P`) actually opens a port. *(Part 5 — a favorite trap)*

**27. My app runs but I cannot reach it on the published port. Why?**
Most often the app listens on `127.0.0.1` inside the container, which is unreachable from outside. It must listen on `0.0.0.0`. Then check that `-p` is set, the internal port is right, and the firewall allows it. *(Part 10)*

**28. `docker logs` shows nothing. Why?**
The app writes to a log file inside the container instead of printing to the screen. Containers should log to standard output and let the platform collect it. *(Part 4)*

**29. How do restart policies work?**
`no`, `on-failure[:n]`, `always`, `unless-stopped`. `always` also restarts after a server reboot; `unless-stopped` respects a manual stop. On one server, this is your whole availability strategy. *(Part 4)*

**30. How does port publishing work internally?**
Docker writes a Linux firewall (iptables) rule that forwards traffic arriving at the host port to the container's IP and port. Containers get outbound access through address translation. *(Part 6)*

**31. What is a healthcheck and where is it used?**
A command Docker runs periodically to decide whether the app is actually working. It drives the `healthy` status, Compose's startup ordering, and zero-downtime deploys. *(Parts 5, 8, 14)*

**32. How do you copy files in and out of a container?**
`docker cp`. And `docker diff` shows what a container has changed relative to its image. *(Part 4)*

**33. How do you limit a container's resources, and why must you?**
`--memory`, `--cpus`, `--pids-limit`. Without a memory limit, one leaking container can exhaust the server and Linux may kill *any* process to survive — including your other apps or SSH. *(Part 4)*

**34. What are the four network modes?**
Bridge (default, private with published ports), user-defined bridge (adds name resolution — use this), host (shares the host's network directly), and none. *(Part 6)*

**35. Difference between `docker compose down` and `down -v`?**
`down` removes containers and the network but keeps volumes. `-v` also deletes the volumes and your data. *(Part 8)*

## Advanced (36–52)

**36. What happens when a container writes to a file that came from the image?**
Copy-up: the entire file is copied into the writable layer first, then modified. This is why write-heavy data should live on a volume rather than the container layer. *(Part 3)*

**37. I put a secret in one layer and deleted it in the next. Am I safe?**
No. The earlier layer still contains it and anyone can extract it from the image. Use build-time secret mounts, or inject at runtime. *(Parts 3, 11)*

**38. Is root inside a container the same as root on the host?**
By default it is the same user ID, separated only by dropped capabilities, syscall filtering, and namespaces. Combined with a host bind mount or socket access it is effectively host root. Mitigate with a non-root user, user namespaces or rootless mode, and dropping capabilities. *(Parts 2, 11)*

**39. A container exits with 137 at 3 AM. Walk me through it.**
137 means killed by force. Check `OOMKilled`: if true, it exceeded its memory limit — look at the memory trend to distinguish a leak from a spike, then fix or raise the limit. If false, something else killed it, or it ignored the shutdown signal until the timeout expired. *(Part 10)*

**40. How do you debug a container that has no shell?**
Join its namespaces from outside — run a tools container with `--pid container:X --network container:X`, or use `nsenter` from the host. You never actually need a shell inside the image. *(Part 10)*

**41. Why did Kubernetes "drop Docker," and why did nothing break?**
It removed the adapter to the Docker *daemon* and now talks directly to containerd — the same runtime inside Docker. Images follow an open standard, so build workflows were unaffected. *(Part 15)*

**42. Design an image tagging strategy.**
Immutable tags produced by CI (git commit ID plus semantic version), production pinned to digests, `latest` never deployed. This makes "what is running?" and "roll back" answerable with certainty. *(Part 9)*

**43. Explain "build once, promote everywhere."**
Build the image a single time; the same digest moves through every environment. Rebuilding per environment produces an artifact that was never tested. Environment differences go in configuration. *(Part 12)*

**44. Our CI builds are slow. What do you do?**
CI machines are wiped between runs, so add an explicit cache backend (`--cache-from`/`--cache-to`, `mode=max`), fix Dockerfile ordering, add cache mounts for package managers, and consider persistent remote builders. *(Part 12)*

**45. How do multi-architecture images work?**
One tag points to a list of images, one per CPU type; clients pull the matching one. Build with `buildx --platform`. Emulation is slow — cross-compiling or native builders is the fast path. *(Parts 9, 12)*

**46. How do you build images inside a containerized CI job?**
Four options: mount the host socket (fast, but the job gets host root), Docker-in-Docker (isolated, but privileged and cold cache), daemonless rootless builders (the secure choice), or remote builders (fastest, warm cache, native multi-arch). *(Part 12)*

**47. Harden a container at runtime — go.**
Non-root user, read-only filesystem plus a small tmpfs, drop all capabilities and add back only what is needed, no-new-privileges, resource and process limits, and run a pinned digest. Escalate to rootless/user namespaces, then sandboxed runtimes for untrusted code. *(Part 11)*

**48. A developer says the app only works with `--privileged`. Your response?**
`--privileged` disables essentially all container protections. Find the actual requirement — usually one capability or one device — and grant only that. *(Part 11)*

**49. Where do secrets live at build time and at run time?**
Build: secret mounts only, never ARG/ENV/COPY. Run: injected at start from a secret store or environment, preferably as files. Remember `docker inspect` exposes environment variables in plain text. *(Part 11)*

**50. What is an SBOM and why does it matter?**
A machine-readable list of everything inside an image. When a major vulnerability is announced, "which of our images contain this?" becomes a query instead of days of investigation. *(Part 11)*

**51. What does image signing prove?**
Who built the image — not that it is safe. It defends against someone with stolen registry credentials pushing a malicious image. Keyless signing uses the CI system's identity, so there is no key to leak. *(Part 11)*

**52. Our scanner passed on Tuesday; the image is critical on Thursday. What happened?**
A new vulnerability was published for a package you already ship. CI-time scanning cannot catch this — you need continuous re-scanning in the registry plus an SBOM inventory, and ideally golden base images so the fix lands in one place. *(Part 11)*

## Scenario-based (53–63)

**53. Every deploy drops requests for about ten seconds.**
The app never receives the shutdown signal — usually shell-form CMD, or no signal handler. Fix the form, implement graceful shutdown, set an adequate stop timeout, and have the load balancer stop sending traffic first. *(Parts 4, 5, 14)*

**54. The server rebooted and containers did not come back.**
The restart policy was `no` or `on-failure`. Use `unless-stopped`/`always`, or run containers as systemd units — and note this gap is one reason orchestrators exist. *(Part 4)*

**55. CI takes 12 minutes, mostly reinstalling dependencies.**
`COPY . .` sits before the install step, so every commit invalidates the cache. Copy the dependency file first, install, then copy the source. Add a CI cache backend. *(Parts 5, 12)*

**56. "Our Postgres container lost all its data after a redeploy."**
The data was in the container's writable layer with no volume mounted — or someone ran `down -v`. Mount a named volume at the data directory and document the difference. *(Parts 7, 8)*

**57. The image works on Intel staging but crashes instantly on new ARM servers.**
It was built for one CPU architecture only. Build a multi-architecture image and verify with `imagetools inspect`. *(Parts 9, 12)*

**58. Small requests work; large uploads hang forever.**
Almost always an MTU mismatch — the host is on a VPN or overlay with smaller packets than Docker assumes. Set the MTU in the daemon config. *(Parts 6, 13)*

**59. The app is slow but the server's CPU looks idle.**
CPU throttling — the container is hitting its `--cpus` limit, which does not show up in host-level CPU usage. Check throttled-period metrics. *(Part 13)*

**60. The production disk is full.**
Check with `docker system df -v`. Usual order: unrotated container logs, old images, orphaned volumes, build cache. Fix log rotation permanently in the daemon config and schedule safe pruning. *(Parts 10, 13)*

**61. Design zero-downtime deploys without Kubernetes.**
A proxy that stays up (Traefik/nginx), start the new version alongside the old, wait for its healthcheck, shift traffic, drain and stop the old one. Plus digest-based rollback and expand-and-contract database migrations. *(Part 14)*

**62. What is genuinely hard about zero-downtime that no tool fixes?**
Database schema changes and real readiness semantics. Expand-and-contract migrations are application discipline, not a platform feature. *(Part 14)*

**63. The security team asks you to prove nothing runs as root.**
Check `docker inspect -f '{{.Config.User}}'` across the fleet, enforce `USER` in Dockerfiles with a CI policy check, and discuss user namespaces and rootless mode as defense in depth. *(Part 11)*

## Trick and gotcha questions (64–70)

**64. "Containers are more secure than VMs, right?"**
No — less isolated by default, because the kernel is shared. That is exactly why cloud providers run untrusted containers inside micro-VMs. *(Parts 11, 15)*

**65. "EXPOSE opens the port, doesn't it?"**
No. It is documentation. Only `-p`/`-P` publishes. *(Part 5)*

**66. "`docker compose down` deletes my data."**
Only with `-v`. Both directions of this get asked. *(Part 8)*

**67. "A stopped container is gone."**
No — its filesystem, logs, and configuration remain until `docker rm`, and they consume disk. *(Part 4)*

**68. "An Ubuntu container runs the Ubuntu kernel."**
No — always the host's kernel. `uname -r` inside any container proves it. *(Part 15)*

**69. "Alpine is always the best base image."**
Not always. Its different C library breaks some compiled packages and changes DNS behavior. Slim Debian or distroless is often the better trade. *(Part 5)*

**70. "`latest` means the newest version."**
No. It is just a default label that can point at anything, including something old. Never deploy it. *(Part 9)*

---

# Part 17 — Hands-on labs

Reading gets you nowhere. These labs are the actual learning. Each maps to a part of the guide — if you get stuck, that is the part to re-read.

## Lab 1 — See that a container is just a process (30 min) → Part 2

```bash
docker run -d --name web nginx
docker exec web ps aux                    # inside: nginx is process 1
ps aux | grep 'nginx: master'             # on the host: same process, different number
docker inspect -f '{{.State.Pid}}' web    # its real process number on the host
sudo ls /proc/$(docker inspect -f '{{.State.Pid}}' web)/ns/    # its namespaces
docker exec web uname -r && uname -r      # SAME kernel — proof there is no guest OS
```

## Lab 2 — Explore images and layers (20 min) → Part 3

```bash
docker history python:3.12-slim           # see every layer and what created it
docker run -d --name t python:3.12-slim sleep 999
docker exec t touch /tmp/hello
docker diff t                             # what changed in the writable layer
docker rm -f t                            # and now it is gone forever
docker system df -v
```

## Lab 3 — Build a real production image (60 min) → Part 5

Take any small app you have written.

1. Write the naive Dockerfile. Build it. Note the size and build time.
2. Change one line of source code and rebuild. Time it. (Painful, right?)
3. Now write the production version: multi-stage, dependency file copied first, non-root user, bracket-form CMD, `.dockerignore`, healthcheck.
4. Rebuild after a source change. Compare the time and the image size.
5. Deliberately `COPY` a fake secret file and delete it in a later step, then find it with `docker history` and `docker save`. **See the leak with your own eyes** — you will never forget it.

## Lab 4 — Networking and storage (40 min) → Parts 6, 7

```bash
docker network create appnet
docker volume create pgdata
docker run -d --name db --network appnet -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret postgres:16
docker run --rm -it --network appnet nicolaka/netshoot getent hosts db   # name resolves!
```

Then prove data survives: create a table, `docker rm -f db`, start a new container with the same volume, confirm the table is still there. Finally, make your app listen on `127.0.0.1`, watch the published port fail, and fix it to `0.0.0.0`.

## Lab 5 — Break things on purpose (40 min) → Part 10

This lab makes you unafraid of production.

```bash
# Exit 137 — out of memory
docker run --memory=32m python:3.12-slim python -c "x='a'*10**9"
docker inspect -f '{{.State.OOMKilled}}' $(docker ps -lq)

# Exit 127 — command not found
docker run alpine bash            # alpine has no bash!

# A crash loop
docker run -d --restart always --name crasher alpine sh -c "exit 1"
docker ps -a                      # watch the restart count climb
docker rm -f crasher

# Log rotation: fill the disk with logs, then fix it permanently in daemon.json
docker run -d --name noisy alpine sh -c "while true; do echo spam; done"
sudo du -sh /var/lib/docker/containers/*/*-json.log
docker rm -f noisy
```

## Lab 6 — Compose a real stack (30 min) → Part 8

Build the app + Postgres stack from Part 8. Then: break the database password and read the failure in the logs; fix it; verify `down` keeps your data and `down -v` destroys it; add a `debug` profile with a database admin UI.

## Lab 7 — A real CI pipeline (90 min) → Part 12

Push your app to GitHub with the workflow from Part 12. Then:

- Confirm caching works (compare the build step time on run 1 vs run 2).
- Deliberately use an old vulnerable base image and watch the scan **block** the pipeline. Fix it.
- Add `--sbom=true`, then inspect the SBOM and find one dependency you did not know you were shipping.

## Lab 8 — Harden and operate (60 min) → Parts 11, 13

1. Run your image with the full hardened flag set from Part 11. Fix what breaks (usually a write path or a low port — each fix is one line).
2. Write a `daemon.json` with log rotation, `live-restore`, metrics, and a custom address pool. Restart Docker and **confirm your containers survived** — that is `live-restore` proving itself.
3. Run cAdvisor + Prometheus + Grafana with Compose, graph your app's memory, then trigger the memory kill from Lab 5 and find it on the graph.

## Lab 9 — Zero-downtime deploy (45 min) → Part 14

Put Traefik in front of your app. Script the deploy: start version 2, wait for healthy, shift traffic, drain and stop version 1. **Prove it works**: run a load generator (`hey` or `ab`) continuously during the deploy — zero failed requests is a pass. Then remove the healthcheck, deploy again, and watch errors appear. That contrast is the whole lesson.

---

# You are done — the final self-check

Tick these honestly. Anything unchecked tells you exactly what to review.

**Fundamentals**

- [ ] I can explain what Docker is by describing the problem first, not the definition
- [ ] I can explain why a container is a process, not a small virtual machine
- [ ] I can narrate what happens during `docker run` from the CLI down to runc
- [ ] I can draw the layer diagram and explain copy-up, hidden files, and the deleted-secret trap

**Building**

- [ ] I can take a bad Dockerfile and fix all six common problems, explaining each
- [ ] I can explain layer caching and order a Dockerfile correctly
- [ ] I can write a multi-stage build and say how much it saves
- [ ] I know why bracket-form CMD matters for shutdown

**Operating**

- [ ] I know the connection rules: container-to-container vs host-to-container
- [ ] I can choose correctly between volume, bind mount, and tmpfs
- [ ] I can diagnose exit codes 0, 1, 126, 127, 137, 139, 143 from `docker ps -a`
- [ ] I can debug a container that has no shell
- [ ] I can write a `daemon.json` and explain what each setting prevents

**Production**

- [ ] I can list the hardening flags with a reason for each, and say why `--privileged` is a red flag
- [ ] I can explain where secrets live at build time and at run time
- [ ] I can design a CI/CD pipeline and defend "build once, promote everywhere"
- [ ] I can design a zero-downtime deploy without Kubernetes, including database migrations
- [ ] I can explain Docker vs VMs vs Kubernetes at all three depths

**When every box is ticked, you are ready — for the job and for the interview.**

---

# Command cheat sheet

```
IMAGES                                        CONTAINERS
docker build -t name:tag .                    docker run -d --name X -p 8080:80 img
docker images                                 docker ps [-a]
docker tag src dst                            docker stop / start / restart / kill X
docker push / pull name:tag                   docker rm [-f] X
docker history img                            docker logs -f --tail 100 X
docker inspect img                            docker exec -it X sh
docker rmi img  |  docker image prune -a      docker inspect X
docker save / load       (move without a      docker stats  |  docker top X
                          registry)           docker cp X:/path ./  |  docker diff X

BUILDING (advanced)                           NETWORKS
docker buildx build --platform ...            docker network create net
  --cache-from / --cache-to ...               docker network ls / inspect net
  --sbom=true --provenance=true               docker run --network net ...
docker buildx bake                            docker run --network container:X ...
docker buildx imagetools inspect img
                                              VOLUMES
COMPOSE                                       docker volume create / ls / inspect / rm
docker compose up -d [--build] [--watch]      docker volume prune
docker compose ps  |  logs -f  |  exec S sh
docker compose down       (keeps volumes)     HOST AND DAEMON
docker compose down -v    (DELETES data)      docker system df -v
docker compose config     (merged view)       docker system prune -af --filter until=168h
docker compose --profile debug up             docker events  |  info  |  version
                                              docker context create X --docker "host=ssh://..."
DEBUGGING ONE-LINERS
docker inspect -f '{{.State.OOMKilled}}' X    SWARM (awareness only)
docker inspect -f '{{.State.Pid}}' X          docker swarm init / join
docker run --rm -it --pid container:X \       docker service create / ls / ps
   --network container:X nicolaka/netshoot    docker stack deploy -c compose.yaml app
```

**Exit codes:** 0 finished · 1 app error · 126 not executable / wrong CPU · 127 not found · 137 killed (usually out of memory) · 139 crashed · 143 stopped politely

---

**Next topic in your DevOps journey: Kubernetes.** Look at the table in Part 15 — you have already learned most of its core concepts under different names. It will feel like naming things you already understand.
