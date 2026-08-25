# Docker: Zero to Production Expert
 
 
## How to use this guide

This guide takes you from "I have never used Docker" to "I can run Docker in production and defend my choices in an interview."

It is written in simple language on purpose. Every technical word is explained the first time it appears. But the *content* is not simplified — you will learn the same things a senior DevOps engineer knows, just explained clearly.

**The path:**

```
 Part 0-1   Understand + install               (1 hour)   <- absolute beginner
 Part 2-4   How Docker really works            (3 hours)  <- the "aha" section
 Part 5     Writing Dockerfiles                (2 hours)  <- most valuable skill
 Part 6-9   Networking, storage, Compose       (3 hours)
 Part 10    Fixing things when they break      (2 hours)  <- job skill #1
 Part 11-14 Security, CI/CD, running servers   (4 hours)  <- production ready
 Part 15    Docker vs VMs vs Kubernetes        (30 min)
 Part 16    70 interview questions             (2 hours)
 Part 17    Hands-on labs                      (do these!)
```

**Two rules for getting real value:**

1. **Type every command.** Reading about Docker teaches you nothing. Ten minutes of typing beats an hour of reading.
2. **Do the labs in Part 17.** They are where the knowledge sticks.

**Symbols used in this guide:**

- 🔴 **Mistake** — something people get wrong all the time
- 🎯 **Interview** — this exact thing gets asked
- 🏭 **Production** — what changes when real users depend on it
- ⚙️ **Depends on** — behavior that changes by operating system or Docker version

**Current as of 2026:** Docker Engine 29.x, Compose v2, BuildKit. Linux is the focus, because that is what servers run. Mac and Windows differences are called out where they matter.

---

# Part 0 — What Docker is and why it exists

## The problem, told as a story

You write an app on your laptop. It works. You send it to a server. It crashes.

Why? Because your app is not just your code. Your app is:

```
   your code
 + the language runtime      (Python 3.12, Node 22, Java 21...)
 + system libraries          (OpenSSL, glibc, image libraries...)
 + config files and settings
 + environment variables
 + the right folder layout
```

Your laptop has all of that set up correctly, because you set it up over months and forgot half of it. The server does not. So the app breaks, and everyone says the famous sentence: *"But it works on my machine."*

## What Docker does about it

Docker lets you put **all of that** — your code plus everything it needs to run — into a single package.

That package is called an **image**. You can copy it anywhere. When you run it, it behaves the same everywhere: on your laptop, on your teammate's laptop, in the test system, on the production server.

A running copy of an image is called a **container**.

```
   IMAGE                              CONTAINER
   (the package, sitting still)       (the package, running)

   Like a .zip you can run            Like the program after you started it
   Read-only, never changes           Has its own temporary workspace
   One image ->                       -> many containers at once
```

That is the whole idea. Everything else in this guide is detail.

## Why this changed how software is deployed

Before Docker, "deploy" meant: copy files to a server, install packages, edit config, restart, hope. Every server slowly became unique, hand-tuned, and impossible to recreate. People called them "snowflake servers."

After Docker, "deploy" means: pull image `myapp:1.4.2`, run it. And critically:

| Before Docker | With Docker |
|---|---|
| Deploy = many manual steps | Deploy = run this image |
| Test and production drift apart | Same image runs in both |
| Rollback = restore backups, reinstall | Rollback = run the older image |
| One app per server (they conflict) | Many apps per server (isolated) |
| Servers configured by hand | Servers are disposable |

🎯 **Interview:** When asked "what is Docker," do not recite a definition. Tell the problem first: *"Docker packages an application together with everything it needs to run, so the exact same package runs identically on a laptop, in CI, and in production. That removes environment drift, which was the single biggest cause of deployment failures."* Then add the senior point: **the image becomes the unit of deployment** — CI/CD pipelines, rollbacks, and Kubernetes all exist on top of that one idea.

## The shipping container comparison (and why it is actually good)

Real shipping containers changed global trade. Not because the box is clever — because it is *standard*. Cranes, ships, and trucks do not care what is inside. They only know how to move a standard box.

Docker did the same for software. Your CI system, your servers, and Kubernetes do not care whether your container holds Python, Java, or Go. They only know how to run "a container." That standardization is the real win.

---

# Part 1 — Install Docker and run your first container

## Install on Linux (Ubuntu / Debian)

Use Docker's official package source. The version in Ubuntu's own repository is usually old.

```bash
# 1. Let apt trust Docker's package signing key
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# 2. Add Docker's repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Install Docker plus its plugins
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
     docker-buildx-plugin docker-compose-plugin

# 4. Check it works
sudo docker run --rm hello-world
```

### Run Docker without typing sudo every time

```bash
sudo usermod -aG docker $USER    # add yourself to the "docker" group
# log out and back in, then:
docker ps
```

🔴 **Mistake / 🏭 Production:** People treat this as a harmless convenience. It is not. Being in the `docker` group means you can ask Docker to start a container that has full access to the whole machine. **Docker group access = admin access to that server.** On your laptop, fine. On a shared or production server, treat it exactly like giving someone `sudo`. (Part 2 explains why this is true.)

## Install on Mac or Windows

Install **Docker Desktop** (alternatives: OrbStack or Colima on Mac).

⚙️ **Important thing to understand:** Containers are a Linux feature. Mac and Windows do not have a Linux kernel. So Docker Desktop quietly runs a small Linux virtual machine in the background, and all your containers actually run inside that hidden VM.

This explains three things that confuse beginners:

- Sharing folders from your Mac into containers is slower than on Linux.
- Some networking options behave differently.
- If you have an Apple Silicon Mac, your images are built for ARM by default and may not run on an Intel/AMD server. (Part 12 shows the fix.)

## Your first containers

```bash
docker run hello-world
```

What happened: Docker looked for the `hello-world` image locally, did not find it, downloaded it from Docker Hub (the public image library), started a container from it, the container printed a message and exited.

Now something more useful:

```bash
docker run -d -p 8080:80 --name web nginx
```

Open http://localhost:8080 — a real web server is running. You did not install nginx. You did not configure anything.

Breaking that command down piece by piece:

| Part | Meaning |
|---|---|
| `docker run` | Create a container and start it |
| `-d` | "Detached" — run in the background, give me my terminal back |
| `-p 8080:80` | Connect port 8080 **on my machine** to port 80 **inside the container** |
| `--name web` | Call it "web" so I can refer to it later instead of using a random ID |
| `nginx` | The image to run |

Now look around:

```bash
docker ps                    # what is running
docker logs web              # what has it printed
docker exec -it web bash     # get a shell INSIDE the container
  ls /usr/share/nginx/html   #   (run this inside; it is a whole little Linux)
  exit
docker stop web              # stop it
docker rm web                # delete it
```

Take a moment on what just happened. You ran a full web server, opened a shell inside its own private filesystem, and deleted the whole thing — in about 20 seconds, without touching your real system.

## The three words you must not mix up

Beginners lose hours to this confusion, so let us kill it now:

```
  DOCKERFILE  --build-->  IMAGE  --run-->  CONTAINER

  a recipe                a package         a running thing
  (text file              (built once,      (started from an image,
   you write)              reused many       can run many at once)
                           times)
```

- **Dockerfile** — a text file with instructions for building an image. You write it.
- **Image** — the built result. Read-only. Never changes.
- **Container** — an image that is running (or stopped but not deleted). Has its own scratch space.

🎯 **Interview:** "Image vs container?" Answer: *"An image is the immutable build artifact — like a class in code. A container is a running instance of it, with its own writable layer and its own runtime state — like an object. You can run many containers from one image."*

---

# Part 2 — How Docker actually works inside

This part is what separates people who *use* Docker from people who *understand* it. It is also the most common source of senior interview questions. Take it slowly.

## The big surprise: a container is just a process

Beginners imagine a container is a tiny virtual machine. **It is not.**

A container is a normal Linux process — the same kind of thing as your text editor or web browser — that the operating system is deliberately lying to.

The lies are of three kinds:

1. **Namespaces** decide *what the process can see.*
2. **Cgroups** (control groups) decide *how much the process can use.*
3. **A layered filesystem** gives it *its own private view of the disk.*

That is it. No small operating system inside. No virtual hardware. Just a process wearing a costume.

**Prove it to yourself right now:**

```bash
docker run -d --name demo nginx
docker exec demo ps aux            # inside: nginx is process number 1
ps aux | grep 'nginx: master'      # on your host: the SAME process, different number
```

The nginx process is visible on your host machine, because it *is* a host process. It just does not know it.

### Namespaces — what the process can see

Each namespace hides one part of the system:

| Namespace | Hides | What the container experiences |
|---|---|---|
| `pid` | Other processes | "I am process 1. I see no other programs." |
| `net` | The network | "I have my own network card, my own IP, my own ports." |
| `mnt` | Filesystems | "I have my own root folder `/`. There is no host filesystem." |
| `uts` | Hostname | "My machine name is a4f9c2b1." |
| `ipc` | Shared memory | "I cannot see other programs' shared memory." |
| `user` | User IDs | (optional) "I am root here" — while being an ordinary user outside |

### Cgroups — how much the process can use

Cgroups are limits enforced by the Linux kernel:

```bash
docker run -d --memory=512m --cpus=1.5 nginx
```

If the container tries to use more than 512 MB of memory, the kernel **kills it**. (You will meet the resulting exit code 137 many times in Part 10 — remember this moment.)

⚙️ **Depends on:** Modern Linux uses **cgroups v2** (Ubuntu 22.04+, RHEL 9+). Very old systems used v1, where memory limits sometimes silently failed to apply. If you meet a machine where limits "don't work," check this.

## The components: Docker is not one program

When you type `docker`, you are talking to a stack of separate programs. Knowing this stack is a strong senior signal, and it explains a famous Kubernetes headline (Part 15).

```
  YOU TYPE:  docker run nginx
      |
      v
  +--------------------------------------------------+
  |  docker  (the command you type)                   |
  |  Just a client. It sends a web request and waits. |
  +--------------------------------------------------+
      |  sends a request over a local socket file
      |  (/var/run/docker.sock)
      v
  +--------------------------------------------------+
  |  dockerd  (the Docker daemon — the background     |
  |  service). Handles the API, builds images,        |
  |  manages networks and volumes.                    |
  |  NOTE: it does not run containers itself!         |
  +--------------------------------------------------+
      |
      v
  +--------------------------------------------------+
  |  containerd  (the container manager)              |
  |  Downloads images, tracks containers, supervises  |
  |  them. Kubernetes uses this same program directly.|
  +--------------------------------------------------+
      |  starts one small helper per container
      v
  +--------------------------------------------------+
  |  containerd-shim  (one per container)             |
  |  A tiny babysitter process that stays alive as    |
  |  the container's parent.                          |
  +--------------------------------------------------+
      |
      v
  +--------------------------------------------------+
  |  runc  (the thing that actually builds the        |
  |  container: namespaces + cgroups + start)         |
  |  Then runc EXITS. It does not stay running.       |
  +--------------------------------------------------+
      |
      v
   your nginx process, isolated
```

**Why the shim exists** (a great detail to know 🏭): because the little shim — not `dockerd` — is the container's parent, you can **restart or upgrade Docker itself without killing running containers**. In production you turn this on with `"live-restore": true` (Part 13). That is how you patch Docker on a busy server without taking the apps down.

## What `docker run nginx` does, step by step

🎯 **Interview:** Being able to narrate this cleanly puts you ahead of most candidates. Say it in your own words:

1. **You → daemon.** The `docker` command sends "create a container" and "start it" to `dockerd` through a socket file.
2. **Find the image.** Is `nginx` on this machine? If not, containerd downloads it from Docker Hub: first a small list of parts (the *manifest*), then each part (each *layer*) in parallel. Every part is checked against a cryptographic fingerprint so corrupted or tampered downloads fail.
3. **Build the filesystem.** The read-only layers are stacked, and one empty writable layer is placed on top. That stack becomes the container's `/`. (Part 3 explains layers properly.)
4. **Build the network.** Docker creates a virtual network cable: one end plugs into a virtual switch on the host called `docker0`, the other end becomes `eth0` inside the container. The container gets an address like `172.17.0.2`.
5. **Write the recipe.** containerd writes a standard JSON description — which namespaces, which limits, which command — and starts a shim, which calls `runc`.
6. **Create the isolation.** `runc` creates the namespaces, applies the cgroup limits, drops dangerous privileges, applies a **seccomp** filter (which blocks about 44 risky kernel calls), switches into the new root filesystem, and finally starts your program.
7. **runc exits.** Your program is now process number 1 inside its own little world, watched over by the shim. Docker reads its output for `docker logs`.

## Why `/var/run/docker.sock` is dangerous

That socket file is how you talk to `dockerd`. And `dockerd` runs as root.

Anyone who can write to that socket can ask Docker to start a container that mounts the entire host disk and runs with full privileges. In other words:

> **Access to the Docker socket = root access to the machine.**

This is why the `docker` group is not a casual thing (Part 1), why mounting the socket into a container (common for CI tools, Portainer, Traefik) is a real risk you must consciously accept, and why "daemonless" tools like Podman and rootless Docker exist (Part 15).

---

# Part 3 — Images and layers

## Images are built in layers

An image is not one blob. It is a stack of read-only layers, each one recording the changes made by one build step.

```
   A RUNNING CONTAINER
   +------------------------------------------+
   | writable layer   <- everything the        |  Deleted when the
   |                     container writes      |  container is deleted!
   +------------------------------------------+
   | layer 4:  copy in your application code   |  \
   | layer 3:  install your dependencies       |   |  THE IMAGE
   | layer 2:  install python                  |   |  read-only, shared
   | layer 1:  the base operating system files |   |  between containers
   +------------------------------------------+  /
        the container sees all of these merged into one filesystem
```

When the container reads a file, Linux gives it the copy from the **highest layer that has it**.

## Three behaviors you must know

**1. Writing to an existing file copies the whole file up.**

If your container edits a 2 GB file that came from the image, Linux first copies all 2 GB into the writable layer, then edits it there. This is called **copy-up**.

🏭 **Production meaning:** never keep a database's data files in the container's writable layer. Use a volume (Part 7). Otherwise writes are slow and the data dies with the container.

**2. Deleting a file does not remove it from the image.**

Deleting just adds a "this file is hidden now" marker in the top layer. The file still exists in the lower layer.

🔴 **Mistake / 🎯 Interview trap:** Putting a password or key in one build step and deleting it in the next does **not** make it safe. Anyone can unpack the image and read it. The correct way to handle build-time secrets is in Part 11.

**3. Identical layers are stored only once.**

Every layer is named by a fingerprint of its contents. If 20 of your images are built on the same base, that base is stored **once** on disk and downloaded **once**. So "500 MB image × 20 containers" does not mean 10 GB. This also makes pushes and pulls fast — only the parts the other side lacks travel.

## Tags vs digests (this one bites teams)

```bash
nginx:1.27                       # a tag  — a NAME that can be moved to point elsewhere
nginx@sha256:ab3f9c...           # a digest — the exact content, can never change
```

A tag is like a sticky note. Someone can move it to a different image tomorrow. A digest is like a fingerprint — it always identifies exactly one image.

🏭 **Production:** For anything that matters, deploy by digest or by a tag your CI creates once and never reuses. Details and the full tagging strategy are in Part 9.

## Look inside images yourself

```bash
docker pull python:3.12-slim
docker history python:3.12-slim    # every layer and which build step made it
docker inspect python:3.12-slim    # full details: settings, env vars, layer list
docker images                      # what you have locally
docker system df -v                # how much disk images/containers/volumes use
```

⚙️ **Depends on version:** Docker is moving to a newer internal image store (the "containerd image store"), already default in Docker Desktop and available on servers. It handles multi-architecture images better. Day-to-day commands are unchanged — but mentioning this shows you follow current Docker.

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

# Part 11 — Security

Security is not a separate topic bolted on at the end. It is a set of small habits. Here they are, in order of value.

## The one-line summary of the risk

Containers share the host's Linux kernel (Part 2). So the wall between a container and your server is thinner than the wall around a virtual machine. Everything below is about making that wall thicker.

## The hardened run command

```bash
docker run -d \
  --user 10001:10001 \                         # 1. not root
  --read-only \                                # 2. the filesystem cannot be modified
  --tmpfs /tmp:rw,noexec,size=64m \            #    (with a small writable scratch area)
  --cap-drop ALL --cap-add NET_BIND_SERVICE \  # 3. remove all special powers, add back one
  --security-opt no-new-privileges \           # 4. block privilege escalation tricks
  --memory=512m --cpus=1 --pids-limit=200 \    # 5. limits are security too
  --restart unless-stopped \
  myapp@sha256:d4e5...                         # 6. run an exact, verified image
```

Why each line, in plain terms:

1. **Not root.** If someone gets in, they arrive as a nobody user instead of an administrator.
2. **Read-only filesystem.** Attackers cannot install anything or modify your code. Your app writes only to the small temp area you allowed.
3. **Capabilities.** Linux splits "root powers" into about 40 separate permissions. Most apps need none of them. Drop everything, then add back only what breaks. (`NET_BIND_SERVICE` is only needed to use ports below 1024 — or just listen on 8080 instead.)
4. **No new privileges.** Stops programs from re-gaining powers through setuid tricks.
5. **Limits.** A container that can consume all memory or spawn unlimited processes can take down the whole server.
6. **Exact image.** Tags can move (Part 9). A digest cannot.

🔴 **The biggest red flag in container security: `--privileged`.** It removes essentially all protection — all capabilities, all devices, no syscall filtering. It is roughly "root on the host." When someone fixes a permissions error by adding `--privileged`, that is not a fix; find the one capability or device actually needed and grant just that.

## Stronger isolation, when you need it

In increasing order of strength:

1. Default protections (already on): capability limits + a syscall filter called **seccomp**.
2. **AppArmor / SELinux** profiles — finer-grained rules about what files and actions are allowed.
3. **Rootless Docker / user namespaces** — root inside the container maps to an ordinary user outside. Now even a full container escape lands somewhere harmless.
4. **gVisor, Kata Containers, Firecracker** — put a real isolation boundary back. This is what cloud providers use to run *your* containers next to *strangers'* containers safely.

🎯 **Interview trap: "Are containers more secure than VMs?"** The correct answer is **no — containers are less isolated by default**, because they share the kernel. That is precisely why AWS runs Lambda and Fargate workloads inside tiny virtual machines: containers for packaging, VMs for isolation. Knowing that combination exists is a strong signal.

## Secrets: never put them in images

| Situation | Wrong (leaks) | Right |
|---|---|---|
| Build needs a private token | `ARG TOKEN` or copying a credentials file | `RUN --mount=type=secret,id=token ...` — the file exists only during that one step and never enters a layer |
| Build needs SSH access | Copying `~/.ssh` into the image | `RUN --mount=type=ssh ...` with `docker buildx build --ssh default` |
| App needs a database password | Baked into the image or committed in the compose file | Provided when the container starts: environment variables from your CI/secret store, a mounted secret file, or the app fetching it from a vault at startup |
| Rotating a password | Rebuild the image | Just restart with a new value — no rebuild needed |

Two extra points 🏭:

- `docker inspect` shows environment variables in plain text to anyone with Docker access. File-based secrets (mounted into `/run/secrets/`) are a step better.
- Do not log the environment. Crash handlers that dump all environment variables are a real and common leak.

## Keeping images free of known vulnerabilities

```bash
trivy image myapp:1.4.2          # or: docker scout cves myapp:1.4.2
```

What to actually do with this 🏭:

- **Make the scan a gate in CI** that fails the build on critical problems (Part 12). A scanner nobody fails on is decoration.
- **Also scan continuously in the registry.** An image that was clean on Tuesday can be critical on Thursday, because new vulnerabilities are discovered in software you already shipped. A CI-only scan can never catch that.
- **Shrink the image.** A distroless image with no shell and no package manager simply has far less that *can* be vulnerable. Minimalism is a security control.
- **Update base images automatically.** Pin versions for reproducibility, then use automated update pull requests. Pinning without updating just freezes your vulnerabilities in place.
- 🏭 **Golden base images:** one team maintains `mycompany/base-python:3.12`, everyone builds on it. Patch once, everyone benefits. This is standard practice at mature companies — worth naming in interviews.

## Proving an image is really yours

Two related ideas that come up in modern interviews:

- **Signing** (with `cosign`) proves *who built* an image. If someone steals your registry password and pushes a malicious image, an unsigned image fails verification. Modern "keyless" signing uses your CI system's identity, so there is no key file to leak.
- **SBOM** (Software Bill of Materials) is a machine-readable ingredient list of everything inside the image. Why it matters: when the next big vulnerability is announced, "which of our 300 images contain that library?" becomes a database query instead of a two-day panic.

Generate both during the build: `docker buildx build --sbom=true --provenance=true ...`

---

# Part 12 — CI/CD: automating build, test, and deploy

## The pipeline, and the one rule that matters

```
 git push
    |
    v
 +-------+   +-------+   +-------+   +-------+   +---------+
 | BUILD |-->| TEST  |-->| SCAN  |-->| PUSH  |-->| DEPLOY  |
 | image |   | in the|   | fail  |   | to    |   | pull by |
 |       |   | image |   | on    |   |registry|  | digest  |
 |       |   |       |   |critical|  |        |  |         |
 +-------+   +-------+   +-------+   +-------+   +---------+
     ^                                                |
     |   BUILD ONCE. The exact same image travels     |
     |   all the way to production.                   |
     +------------------------------------------------+
```

🏭 **The rule: build once, promote everywhere.** Build the image a single time. That same image (same digest) moves through test → staging → production. You never rebuild it per environment.

Why this is not just tidiness: if you rebuild for production, you get a *different* image than the one you tested — new base layers may have appeared, dependencies may resolve differently. You would be deploying something that was never tested. 🎯 "We rebuild for prod" is a red-flag answer in interviews.

Corollary: **differences between environments belong in configuration, not in the image.** Same image, different environment variables.

## A real pipeline (GitHub Actions)

```yaml
name: build-and-ship
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta                              # works out the tags automatically
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=git-              # git-abc1234 — unique and permanent
            type=ref,event=tag                # v1.4.2 when you tag a release

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}   # PRs build but do not publish
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha                # reuse cache between runs
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
          sbom: true
          provenance: true

      - name: Security scan (blocks the build)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:git-${{ github.sha }}
          exit-code: '1'                      # <- this makes it a gate, not a report
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true
```

The same shape works in GitLab CI, Jenkins, or anything else. The lessons that transfer:

- **Pull requests build but do not publish** — you catch broken Dockerfiles without polluting the registry.
- **Tags are automatic, unique, and traceable to a commit.**
- **Caching must be explicit in CI.** 🔴 CI machines are wiped between runs, so without `cache-from`/`cache-to` every build starts cold. Adding it typically takes builds from ~10 minutes to under a minute. Build speed is deploy speed is how fast you can fix an outage.
- **The scan blocks the pipeline** when it finds something critical.

## Testing with containers

- **Unit tests** can run inside a build stage of your Dockerfile, so they test the exact filesystem that ships.
- **Integration tests** can start real dependencies: `docker compose -f compose.test.yaml up --exit-code-from tests`. No more fake databases that behave differently from the real one.
- **Smoke test the built image** before pushing: run it and hit `/health`. Catches a broken `ENTRYPOINT` before the registry ever sees it.

## Building images inside CI (a real puzzle)

Your CI job itself runs in a container. So how does a container build a container? Four answers 🎯:

| Approach | How | Trade-off |
|---|---|---|
| **Mount the Docker socket** | Give the job access to the host's Docker | Fast, shares cache — but the job effectively gets root on the CI server |
| **Docker-in-Docker (DinD)** | Run a second Docker inside the job | Isolated per job — but needs `--privileged`, and starts with a cold cache |
| **Daemonless builders** | Rootless BuildKit, Buildah | No daemon, runs unprivileged. ⚙️ The security-conscious choice. (Kaniko was popular but has been in maintenance mode since 2024 — do not pick it for new work) |
| **Remote builders** | The job sends the work to a persistent build server | Fastest: the cache stays warm, and you get real ARM machines instead of slow emulation |

The trend is away from giving CI jobs a Docker daemon at all.

## Promoting between environments

Promotion moves a *label*, not bytes:

```bash
# Re-tag the exact same image for production — no rebuild, no re-upload
docker buildx imagetools create -t myapp:prod myapp:git-abc1234
```

Later in your DevOps journey, GitOps formalizes this: a git repository records which image digest runs where, and deploying is just a commit.

## Making builds fast (the details that matter)

```bash
docker buildx build \
  --cache-from type=registry,ref=ghcr.io/myteam/myapp:buildcache \
  --cache-to   type=registry,ref=ghcr.io/myteam/myapp:buildcache,mode=max \
  -t ghcr.io/myteam/myapp:git-$SHA --push .
```

- `mode=max` caches intermediate build stages too, not just the final ones. Usually what you want.
- Combine with the Dockerfile ordering from Part 5 — ordering decides *what* gets reused; cache backends decide *where the cache survives*.
- **Cross-CPU builds are slow** because they run through emulation. The fix that impresses interviewers: cross-compile instead. Let the compiler run at native speed and only produce output for the other CPU:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.23 AS build
ARG TARGETARCH
RUN GOARCH=$TARGETARCH go build -o /bin/server .
```

- **Many images in one repository?** `docker buildx bake` builds them all in parallel from one definition file, sharing cache — instead of a homemade shell script.

---

# Part 13 — Running Docker servers in production

Now you own the machines. This is what changes.

## The daemon configuration file

`/etc/docker/daemon.json` is the control panel for a Docker host:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" },
  "live-restore": true,
  "default-address-pools": [ { "base": "10.200.0.0/16", "size": 24 } ],
  "registry-mirrors": ["https://mirror.internal:5000"],
  "metrics-addr": "127.0.0.1:9323",
  "default-ulimits": { "nofile": { "Name": "nofile", "Soft": 65536, "Hard": 65536 } }
}
```

What each line prevents:

- **Log rotation** — 🔴 the number one preventable Docker outage. Without it, a chatty container fills the disk and takes the whole server down. (Applies to newly created containers.)
- **`live-restore: true`** — lets you restart or upgrade Docker without killing running containers (this works because of the shim, Part 2). Essential on production hosts.
- **`default-address-pools`** — 🎯 Docker's default internal network range (172.17.x) frequently collides with company VPNs and cloud networks. The symptom is "everything broke when I connected to the VPN." Set your own range that does not clash. This is painfully-learned knowledge that experienced interviewers recognize instantly.
- **`registry-mirrors`** — a local cache of images. Protects you when Docker Hub is down or rate-limits you, and makes pulls much faster.
- **`metrics-addr`** — Docker exposes its own metrics for monitoring.

Apply changes with `sudo systemctl reload docker` (or restart — safe when `live-restore` is on).

## Managing remote servers safely

```bash
docker context create prod1 --docker "host=ssh://ops@prod1.internal"
docker --context prod1 ps        # your local command, their server, over SSH
```

🔴 **Never expose the Docker API on a plain TCP port** (2375). Automated scanners find open Docker ports within hours and install cryptocurrency miners. This is not theoretical — it is one of the most common cloud breaches. Use SSH contexts.

## Logging in production

```
  your app prints to the screen
        |
        v
  Docker's log driver (json-file, with rotation!)
        |
        v
  a collector agent (Fluent Bit / Promtail / Vector) reads the log files
        |
        v
  a searchable system (Loki, Elasticsearch, CloudWatch)
```

🏭 The "collect from files with an agent" pattern is usually better than shipping logs directly from Docker to a remote system. Why: if the remote system goes down and Docker is configured to send logs directly, **your application can block waiting to write logs**. An agent reading files keeps your app unaffected.

Also: write logs as structured JSON lines. Then "find all errors for customer X yesterday" is a query, not a grep.

## Monitoring

Three levels, and you need all three:

```bash
docker stats                      # live, human — for debugging right now
curl -s localhost:9323/metrics    # Docker's own health metrics
# cAdvisor: per-container CPU/memory/network/disk metrics for Prometheus
docker run -d --name cadvisor -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /sys:/sys:ro -v /var/lib/docker:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor
```

Alert on the things that actually page you: memory approaching the limit (predicts the OOM kill before it happens), restart counts climbing, disk filling on `/var/lib/docker`, and — 🎯 the one people forget — **CPU throttling**.

**Why throttling deserves its own paragraph:** if a container hits its `--cpus` limit, the host's overall CPU graph looks *fine* while your application quietly gets slower. The symptom is "the app is slow but the server looks idle." The metric to check is throttled time (`container_cpu_cfs_throttled_periods_total`). Naming this in an interview marks you as someone who has actually operated containers.

## Routine housekeeping

```bash
# Safe scheduled cleanup: only things unused for a week, and NOT volumes
docker system prune -af --filter "until=168h"
```

🔴 Never schedule `--volumes`. Also watch disk on `/var/lib/docker`, and read release notes before major Docker upgrades.

---

# Part 14 — Deploying without downtime (no Kubernetes needed)

You will not always have Kubernetes. A team with two servers and a Compose file still expects deploys that do not drop users' requests. 🎯 "Design zero-downtime deploys without Kubernetes" is an excellent interview question.

## Why the naive way drops requests

`docker compose up -d` with a new image stops the old container, then starts the new one. In between, nothing is listening. Requests fail. And if your app ignores the shutdown signal (Part 4), in-flight requests are killed mid-response.

## The pattern: a proxy that never restarts

```
                       +---------------------+
   users ---- :443 --->|  proxy (nginx or    |   <- stays up the whole time
                       |  Traefik)           |
                       +----------+----------+
                                  |
                 +----------------+----------------+
                 v                                 v
          app version 1                     app version 2
          (running, serving)                (starting up...)

  Deploy steps:
   1. Start version 2 alongside version 1
   2. WAIT until version 2 reports healthy       <- the healthcheck earns its keep
   3. Send new traffic to version 2
   4. Stop sending traffic to version 1, let it finish current requests
   5. Stop version 1
```

**Traefik** is the easiest modern way: it watches Docker and automatically routes to containers based on labels you put on them. A new healthy container joins the pool by itself.

## The checklist that makes it actually work

Every item is something you learned earlier, now doing its job:

- [ ] A **healthcheck that means "ready for users,"** not just "the process started" (Part 5)
- [ ] The app **handles the shutdown signal**: stop accepting new work, finish what is in progress, exit (Part 4)
- [ ] `--stop-timeout` is longer than your slowest request
- [ ] **Immutable tags/digests**, so rolling back means pointing at the previous image — not rebuilding (Part 9)
- [ ] **Database changes are backward compatible** — the old and new versions both work with the current schema during the overlap
- [ ] The proxy is not restarted as part of the deploy

🏭 **The genuinely hard part is the database.** No tool solves it for you. The technique is called **expand and contract**: first add the new column (both versions work), deploy the new code, then later remove the old column. Never rename or drop something the running version still needs. Saying "expand and contract" in an interview shows real deployment experience.

## What this teaches you about Kubernetes

Every item on that checklist is something Kubernetes automates: readiness probes, rolling updates, graceful termination periods, one-command rollback. You are not going to be surprised by Kubernetes — you will recognize it as a robot doing this checklist for you.

---

# Part 15 — Docker vs VMs vs Kubernetes, and the wider world

## Docker vs virtual machines

```
        VIRTUAL MACHINES                       CONTAINERS

  +---------+  +---------+            +---------+  +---------+
  |  App A  |  |  App B  |            |  App A  |  |  App B  |
  +---------+  +---------+            +---------+  +---------+
  | Libs    |  | Libs    |            | Libs    |  | Libs    |
  +---------+  +---------+            +---------+--+---------+
  | Guest   |  | Guest   |            |   Docker / containerd |
  | OS +    |  | OS +    |            +-----------------------+
  | kernel  |  | kernel  |            |  Host OS — ONE kernel |
  +---------+  +---------+            |  shared by everything |
  |     Hypervisor       |            +-----------------------+
  +----------------------+            |       Hardware        |
  |      Host OS         |            +-----------------------+
  +----------------------+
  |      Hardware        |
  +----------------------+
```

Three levels of answer, so you can match the interviewer's depth 🎯:

**Level 1 (beginner):** A VM simulates a whole computer and runs a full operating system. A container is an isolated process sharing the host's operating system. So containers start in milliseconds instead of minutes and use megabytes instead of gigabytes.

**Level 2 (better):** There is no operating system inside a container. An "Ubuntu container" on a Red Hat server contains Ubuntu's *files and programs* but runs on the *host's* kernel. Run `uname -r` inside any container — you get the host's kernel version.

**Level 3 (senior):** It is a trade-off, not an upgrade.

| | VM | Container |
|---|---|---|
| Isolation | Enforced by hardware/hypervisor | Enforced by the kernel |
| Security | Stronger — a break-in stays inside | Weaker — a kernel flaw can reach the host |
| Can run a different OS/kernel? | Yes | No |
| Startup | Seconds to minutes | Milliseconds |
| How many per server | Tens | Hundreds or thousands |

And the punchline: cloud providers combine both — containers for packaging, tiny VMs (like Firecracker) for isolating different customers.

## Docker vs Kubernetes

They are **different layers, not competitors**:

- **Docker** builds images and runs containers **on one machine**.
- **Kubernetes** decides **which machine out of many** runs each container, and keeps reality matching your intent — restarting things elsewhere when a server dies, scaling copies up and down, rolling out new versions, routing traffic.

Compose stops at one machine. That limit is exactly where Kubernetes starts.

🎯 **The "Kubernetes dropped Docker" question.** Around 2022, Kubernetes removed the adapter that let it talk to the *Docker daemon*. It now talks directly to **containerd** — which, as you know from Part 2, is the same component that sits inside Docker. So:

- Your Docker-built images run on Kubernetes unchanged (they follow an open standard).
- `docker build` remains a perfectly normal way to produce them.
- Only the daemon left the Kubernetes node.

Candidates who say "Kubernetes doesn't support Docker anymore" fail this question. You now know exactly why that is wrong.

**What you have already learned that transfers directly:**

| You learned (Docker) | Kubernetes calls it |
|---|---|
| Image | Image (identical — it is a standard) |
| Container | A container inside a Pod |
| `--network container:X` sharing | How containers in a Pod share networking |
| Restart policy | Deployments and controllers |
| Custom network + name resolution | Services |
| Volumes | PersistentVolumes |
| Healthcheck | Liveness and readiness probes |
| Compose file | Manifests / Helm charts |
| Build once, promote by digest | Exactly the same |

## Docker Swarm (one screen is enough)

Docker's own built-in clustering. It lost to Kubernetes and gets minimal investment now, but you may meet it in small companies, and it is a fair interview question:

- `docker swarm init` on one machine, `docker swarm join` on others.
- You declare services ("run 5 copies of this image") and Swarm places and restarts them.
- `docker stack deploy -c compose.yaml myapp` deploys a Compose file to the cluster.
- It has built-in load balancing, secrets, and rolling updates.
- Manager nodes agree using **Raft** consensus — run 3 or 5, never an even number. (⚙️ Kubernetes uses the same idea in etcd, so this knowledge transfers.)
- **Why Kubernetes won:** ecosystem, managed cloud offerings, extensibility, and community — not core capability.

## The wider container world

```
   Open standards (OCI) define: what an image is, how to run it, how registries work
   -> which is WHY all of these interoperate

   BUILD:     docker build / BuildKit  |  Buildah  |  ko, Jib (no Dockerfile at all)
   RUN:       Docker  |  Podman (no background service)  |  containerd + nerdctl
   LOW LEVEL: runc (default)  |  crun  |  gVisor / Kata (extra-isolated)
   STORE:     Docker Hub  |  GHCR  |  AWS ECR  |  Harbor  |  any standard registry
```

- **Podman** — works almost identically to Docker (`alias docker=podman` mostly works), but has no background service and runs unprivileged by default, which removes the socket risk from Part 2. Default on Red Hat systems.
- **containerd** — what is actually running your containers under Docker *and* under Kubernetes.
- 🎯 **The takeaway for interviews:** because images, runtimes, and registries are open standards, *"Docker skills" are really "container skills."* You can build with one tool, store in another, and run on a third. Nothing you learned here is locked to one vendor.

---

# A few extra things worth knowing

Small topics that do not fit neatly anywhere above, but come up often enough to deserve a home.

## The storage driver has a name: overlay2

When Part 3 described image layers stacking into one filesystem, the Linux feature doing that work is called **overlay2** — it is the default and standard choice today. You will see it in `docker info` output (`Storage Driver: overlay2`) and it is worth recognizing by name in an interview, even though you rarely configure it directly.

## `docker attach` vs `docker exec` — a small but sharp trap

`exec` (which you already use constantly) starts a **brand new** process inside the container — your everyday "open a shell" tool.

`attach` is different: it connects your terminal directly to process 1's own input and output — the process that is already running.

🔴 **The trap:** if you `attach` to see what is happening and then press Ctrl-C to leave, you do not leave — you send an interrupt signal to process 1 itself, and can accidentally **stop the application**. To detach safely without touching the process, use `Ctrl-P` then `Ctrl-Q`.

Rule of thumb: reach for `exec` almost always; only use `attach` when you specifically need to see or interact with process 1's own console.

## Moving an image without a registry: `save`/`load` vs `export`/`import`

Sometimes you need to hand someone an image directly — an air-gapped server, a demo laptop with no internet — without a registry in between.

```bash
docker save -o myapp.tar myapp:1.4.2     # IMAGE -> file: keeps all layers, history, settings
docker load -i myapp.tar                 # restores it identically elsewhere

docker export web -o webfs.tar           # a CONTAINER's current files -> one flat file
docker import webfs.tar mysnapshot:1     # becomes a single-layer image — history and
                                          # settings (CMD, ENV, etc.) are ALL LOST
```

🎯 **The distinction interviewers check:** `save`/`load` is for moving a proper *image* around — nothing is lost. `export`/`import` flattens a *container's* filesystem into a plain tarball, throwing away layer history and configuration. Related: `docker commit` (turning a running container into an image by hand) exists for emergency forensics only — a real image should always come from a Dockerfile you can rebuild, not a container someone tinkered with by hand.

## One Dockerfile instruction we skipped: `ONBUILD`

`ONBUILD RUN ...` adds an instruction that does not run when *this* image is built — it runs later, when **another** image is built `FROM` this one. It is a legacy pattern from early language base images. You are very unlikely to need it, but recognize it if you see it in an old Dockerfile.

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
