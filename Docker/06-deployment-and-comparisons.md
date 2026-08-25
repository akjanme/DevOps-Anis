# Docker Learning — File 6 of 7 — Zero-Downtime Deployment & Docker vs VMs/Kubernetes

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 14 (Zero-downtime deploys) · Part 15 (Docker vs VMs/Kubernetes) · a short extras section
>
> ⬅ Previous: File 5   |   ➡ Next: File 7

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

