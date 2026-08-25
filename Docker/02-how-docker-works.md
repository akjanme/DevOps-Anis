# Docker Learning — File 2 of 7 — How Docker Works Internally

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 2 (Docker's internal architecture) · Part 3 (Images & layers)
>
> ⬅ Previous: File 1   |   ➡ Next: File 3

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

