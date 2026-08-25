# Docker Learning — File 3 of 7 — Running Containers & Writing Dockerfiles

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 4 (Container lifecycle & commands) · Part 5 (Production Dockerfiles)
>
> ⬅ Previous: File 2   |   ➡ Next: File 4

---

# Part 4 — Running and managing containers

## The lifecycle

```
        docker create
             |
             v
  docker run       CREATED
  (create + start)    |  docker start
             \        v
              +--> RUNNING <-------------------+
                     |                          |
        docker stop  |  docker kill             | docker start
     (polite: asks   |  (brutal: kills          |
      it to quit,    |   instantly)             |
      waits 10s,     |                          |
      then kills)    |    program ends by itself|
                     v                          |
                  EXITED  ------------------ ---+
                     |    (still on disk! logs and
                     |     files are still there)
                     |  docker rm
                     v
                  GONE
```

🔴 **Mistake:** thinking a stopped container is gone. It is not. Its files, logs, and settings sit on disk until you run `docker rm`. On busy servers, hundreds of exited containers quietly eat the disk. (`docker ps` shows running ones; `docker ps -a` shows all.)

## The commands you will actually use every day

**Starting and stopping:**

```bash
docker run -d --name web -p 8080:80 nginx     # start in background
docker run -it --rm ubuntu bash               # interactive shell, auto-delete on exit
docker stop web                               # polite stop (10 second grace period)
docker start web                              # start it again
docker restart web
docker rm -f web                              # force delete, even if running
```

**Looking and debugging — you will live in these:**

```bash
docker ps                       # running containers
docker ps -a                    # including stopped ones + why they stopped
docker logs -f --tail 100 web   # see output, follow live
docker exec -it web sh          # open a shell inside a RUNNING container
docker inspect web              # everything Docker knows about it (JSON)
docker stats                    # live CPU and memory usage
docker top web                  # processes inside it
docker cp web:/etc/nginx/nginx.conf ./   # copy a file out
docker diff web                 # what files has it changed?
```

**Cleaning up:**

```bash
docker system df                # what is using disk
docker container prune          # delete all stopped containers
docker image prune -a           # delete unused images
docker system prune -a --volumes   # delete almost everything — know what you type!
```

🔴 **Mistake:** running `docker system prune -a --volumes` on a production host to "free space." That flag deletes volumes — which is where databases keep their data. People have destroyed real data this way. Use targeted cleanup (Part 13).

## Three things that separate beginners from professionals

### 1. `docker logs` only shows output from process number 1

Docker captures whatever your main program prints to the screen (standard output and standard error). That is all.

🔴 **Mistake:** your app writes to `/var/log/app.log` inside the container, so `docker logs` shows nothing, and you conclude Docker is broken.

🏭 **The container rule: print your logs to the screen.** Do not write log files inside containers. Let the platform collect the output. This is why the official nginx image redirects its log files to the screen. Part 13 shows where those logs then go in production.

### 2. Your program becomes process number 1 — and that is special

Linux treats process 1 differently: **default signal handling does not apply to it.**

Practical effect: when you run `docker stop`, Docker politely asks your program to shut down (a SIGTERM signal). If your program does not explicitly listen for that signal, **nothing happens**. Docker waits 10 seconds, then kills it hard.

Every deploy then becomes a hard kill: dropped user requests, half-finished writes.

Two fixes:

```bash
docker run --init ...     # Docker inserts a tiny helper as process 1 that
                          # forwards signals properly and cleans up leftovers
```

or better, handle the shutdown signal in your application code (finish current requests, close database connections, exit). There is also a Dockerfile mistake that causes this — covered in Part 5.

### 3. Restart policies are your safety net

```bash
docker run -d --restart unless-stopped nginx
```

| Policy | Behavior |
|---|---|
| `no` (default) | Container dies, stays dead |
| `on-failure:5` | Restart up to 5 times, only if it crashed |
| `always` | Always restart — including after the server reboots |
| `unless-stopped` | Same, but respects it if *you* stopped it manually |

🏭 On a single server, this is your entire "keep it running" strategy. `unless-stopped` is the usual production choice.

## Always set resource limits in production

```bash
docker run -d \
  --memory=512m \        # hard memory cap
  --cpus=1.5 \           # CPU limit
  --pids-limit=200 \     # max processes (protects against runaway forking)
  myapp
```

🏭 **Why this matters more than it sounds:** without a memory limit, one leaking container can exhaust the whole server's memory. Linux then starts killing processes to survive — and it might kill *any* of them, including your other apps, your database, or the SSH service you need to log in and fix things. A memory limit turns a server-wide outage into one container restarting.

---

# Part 5 — Writing Dockerfiles (the most valuable skill here)

If you learn one thing well, make it this. Bad Dockerfiles cause slow builds, huge images, security holes, and broken deployments. Good ones fix all four.

## How a Dockerfile works

A Dockerfile is a list of build steps. Docker runs them top to bottom. **Most steps create a layer** (Part 3).

```dockerfile
FROM python:3.12-slim        # start from this base image
WORKDIR /app                 # work in this folder from now on
COPY requirements.txt .      # copy a file from your project into the image
RUN pip install -r requirements.txt    # run a command while building
COPY . .                     # copy everything else in
CMD ["python", "app.py"]     # what to run when a container starts
```

Build and run it:

```bash
docker build -t myapp:1.0 .     # the "." means "use this folder as the source"
docker run -d -p 8000:8000 myapp:1.0
```

## Let us fix a bad Dockerfile, one problem at a time

Here is what most tutorials teach:

```dockerfile
FROM python:3.12
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
CMD python app.py
```

It works. It is also wrong in six ways. Let us fix each one, because each fix is a lesson.

### Problem 1: the build is slow every single time

**The rule that governs Dockerfile order:** Docker caches each layer. When you rebuild, Docker reuses a cached layer *unless* that step changed or a file it copies changed. **And once one layer is rebuilt, every layer below it is rebuilt too.**

In the bad example, `COPY . /app` comes first. So every time you change one line of code, that layer changes, and everything below — including installing all your dependencies — runs again. A ten-second change becomes a ten-minute build.

**The fix: put things that rarely change first, things that change constantly last.**

```dockerfile
COPY requirements.txt .              # changes rarely  <- copy the dependency list ALONE
RUN pip install -r requirements.txt  # so this stays cached
COPY . .                             # changes every commit  <- last
```

🎯 **Interview:** "Here is a Dockerfile, optimize it" is a standard exercise. This ordering fix is the first thing they look for.

You can go further with a **cache mount**, which keeps the package downloader's cache between builds without putting it in the image:

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### Problem 2: the image is 1 GB

`python:3.12` is a full Debian system with build tools. You do not need a compiler to *run* a Python app.

Base image size ladder:

| Base | Size | Notes |
|---|---|---|
| `python:3.12` | ~1 GB | Full OS. Fine for building, wasteful for running |
| `python:3.12-slim` | ~120 MB | **Good default.** Normal Linux, minimal extras |
| `python:3.12-alpine` | ~50 MB | Tiny, but uses a different C library (musl). Some packages break or need compiling. Careful. |
| `distroless` / `scratch` | ~2–20 MB | No shell, no package manager. Most secure, hardest to debug |

🔴 **Mistake:** assuming Alpine is always best. Alpine uses `musl` instead of the usual `glibc`. Python packages with compiled parts may fail to install or run slower; DNS behavior differs subtly. For many teams `-slim` is the better trade. Knowing this nuance is a senior signal.

### Problem 3: build tools end up in the final image

**Multi-stage builds** solve this. You build in one stage, then copy only the finished result into a clean, small final stage.

```dockerfile
# ---------- stage 1: build ----------
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# ---------- stage 2: what actually ships ----------
FROM python:3.12-slim
COPY --from=builder /install /usr/local     # take ONLY the installed packages
COPY . .
CMD ["python", "app.py"]
```

For compiled languages the effect is dramatic:

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/server .

FROM gcr.io/distroless/static-debian12    # no shell, no package manager, tiny
COPY --from=build /bin/server /server
USER nonroot
ENTRYPOINT ["/server"]
```

A 1 GB build environment becomes a ~10 MB image. 🏭 Smaller images pull faster (faster deploys, faster scaling) and contain fewer things that can have security holes.

### Problem 4: it runs as root

By default, the user inside a container is **root**. If someone breaks into your app, they are root inside the container — and container root is uncomfortably close to root on the host.

```dockerfile
RUN groupadd -r app && useradd -r -g app app
COPY --chown=app:app . .
USER app
```

🏭 This is the cheapest big security win available. Do it in every image.

### Problem 5: `CMD python app.py` breaks shutdown

There are two ways to write `CMD`, and they behave differently:

```dockerfile
CMD ["python", "app.py"]    # "exec form" — python becomes process 1  ✅
CMD python app.py           # "shell form" — a shell becomes process 1, and
                            # python is its child                       ❌
```

With shell form, `docker stop` sends the shutdown signal to the shell. The shell ignores it. Your app never hears about it. Ten seconds later everything gets killed hard — every deploy drops requests (see Part 4).

🎯 **Rule: always use the bracket form (a JSON array) for `CMD` and `ENTRYPOINT`.**

### Problem 6: unwanted files get copied in

`COPY . .` copies *everything* — your `.git` folder, local `.env` file with passwords, `node_modules`, test data.

Create a `.dockerignore` file next to your Dockerfile:

```
.git
.env
node_modules
__pycache__
*.log
tests/
```

🔴 **Mistake:** shipping secrets into an image through `COPY . .` because there was no `.dockerignore`. Extremely common, and Part 3 explained why deleting them later does not help.

## The production Dockerfile, all fixes applied

```dockerfile
# syntax=docker/dockerfile:1

# ---------- build stage ----------
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

# ---------- runtime stage ----------
FROM python:3.12-slim
RUN groupadd -r app && useradd -r -g app app
WORKDIR /app
COPY --from=builder /install /usr/local
COPY --chown=app:app . .
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
ENTRYPOINT ["python", "app.py"]
```

## ENTRYPOINT vs CMD

- **ENTRYPOINT** = the program that always runs.
- **CMD** = the default arguments (or the whole default command if there is no ENTRYPOINT).
- Anything you type after the image name **replaces CMD**, not ENTRYPOINT.

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

```bash
docker run myapp                 # runs: python app.py --port 8000
docker run myapp --port 9000     # runs: python app.py --port 9000
docker run --entrypoint sh -it myapp   # override the program itself (great for debugging)
```

## HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -fsS http://localhost:8000/health || exit 1
```

This teaches Docker how to tell whether your app is actually working, not just running. `docker ps` will show `healthy` or `unhealthy`. Compose can wait for it before starting other services (Part 8), and it is essential for zero-downtime deploys (Part 14).

⚙️ **Note:** Kubernetes ignores this and uses its own health checks instead. On plain Docker, it matters a lot.

## Other instructions worth knowing

| Instruction | What it does | Watch out |
|---|---|---|
| `WORKDIR /app` | Sets the current folder for later steps | Use this, never `RUN cd /app` (the change is lost) |
| `ENV APP_ENV=prod` | Sets an environment variable, available at build **and** run | Visible to anyone via `docker inspect` — not for secrets |
| `ARG VERSION=1.23` | A variable available **only during build** | An `ARG` written *before* `FROM` cannot be used after it unless re-declared. Classic confusion. Also not for secrets. |
| `EXPOSE 8000` | **Documentation only.** Does not open anything | 🎯 Trap question. Only `-p` actually publishes a port |
| `COPY` vs `ADD` | `COPY` copies files. `ADD` also unpacks tar files and can download URLs | Always use `COPY` unless you specifically want tar extraction |
| `VOLUME /data` | Declares a data folder | Creates surprise anonymous volumes; prefer defining volumes when you run |
| `STOPSIGNAL SIGQUIT` | Change which signal `docker stop` sends | nginx shuts down gracefully on SIGQUIT, for example |
| `LABEL ...` | Attach metadata to the image | Standard labels link images to source repos and help cleanup tools |

## Your Dockerfile checklist

- [ ] Small, specific base image (`-slim` or distroless), version pinned — never `:latest`
- [ ] Dependency files copied *before* application code
- [ ] Multi-stage build, so no compilers ship to production
- [ ] `.dockerignore` exists and excludes `.git`, secrets, local junk
- [ ] `USER` set to a non-root user
- [ ] `ENTRYPOINT`/`CMD` in bracket form
- [ ] App handles the shutdown signal (or you use `--init`)
- [ ] No passwords or keys anywhere in the file
- [ ] `HEALTHCHECK` defined
- [ ] One job per container

---

