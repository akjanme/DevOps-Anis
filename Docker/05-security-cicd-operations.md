# Docker Learning — File 5 of 7 — Security, CI/CD & Running Docker in Production

> Part of the 7-file **Docker: Zero to Production Expert** guide. Content is unchanged from the original single-file version — only split for easier navigation.
>
> Covers: Part 11 (Security) · Part 12 (CI/CD) · Part 13 (Operating Docker hosts)
>
> ⬅ Previous: File 4   |   ➡ Next: File 6

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

