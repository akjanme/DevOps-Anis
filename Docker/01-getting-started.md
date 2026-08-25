# Docker Learning — File 1 of 7 — Getting Started

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 0 (Why Docker exists) · Part 1 (Install & first container)
>
> ⬅ Previous: —   |   ➡ Next: File 2

---

# Docker: Zero to Production Expert

**A plain-English guide. No jargon without explanation. Everything aimed at running Docker for real.**

---

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

