# DevOps Learning Notes — Docker, CI/CD & Kubernetes
*Compiled from whiteboard diagrams with Anis, expanded with explanations and cross-question/answer practice*

---

## 1. Monolithic vs Virtual Machines vs Containers

**What the diagram shows:** A VM box, a "monolithic" label, and a Docker box with "install docker → apache webserver (80) / nginx webserver (80) → single kernel."

### Core idea
- A **monolithic** application is built and deployed as one single, tightly-coupled unit — everything (UI, business logic, data access) ships and scales together.
- A **VM (Virtual Machine)** virtualizes hardware. Each VM has its own full guest OS, kernel, and libraries, running on top of a hypervisor. This makes VMs heavy (GBs in size, slower to boot).
- A **container** (Docker) virtualizes at the OS level. Containers **share the host machine's single kernel** but keep their own filesystem, process space, and network namespace. This is why containers are lightweight and start in seconds, while VMs take minutes.
- Because containers share the kernel, you can run an Apache web server on port 80 and an Nginx web server on port 80 **inside separate containers** on the same host without conflict — each container thinks it owns port 80 internally; the host maps different external ports to each.

### Example commands from the notes
```bash
docker run -d nginx -p 8080:80
# accessible at localhost:8080

docker run -d apache -p 8081:80
# accessible at localhost:8081
```
Both containers listen on port 80 *inside* their own container, but Docker maps them to different host ports (8080 and 8081) so they don't collide.

### Version update note
The board shows `1.0.1 -----> 1.0.2` next to "Update" — a reminder that container images are versioned/tagged, and upgrading an app usually means pulling a new image tag (e.g., `nginx:1.0.2`) and redeploying, rather than patching a running VM in place.

### Cross Q&A
**Q1: Why can two containers both use port 80 internally but not clash?**
A: Each container has its own isolated network namespace, so port 80 inside Container A is not the same as port 80 inside Container B. Only the *host* ports you map with `-p` (8080, 8081) need to be unique on the host machine.

**Q2: What is the single biggest architectural difference between a VM and a container?**
A: A VM includes its own guest OS and kernel on top of a hypervisor; a container shares the host OS's single kernel and only isolates the application layer (process, filesystem, network). This makes containers far lighter and faster to start.

**Q3: If containers share the host kernel, is it safe to run different Linux distros' containers on the same host?**
A: Yes, for the most part — you can run an Ubuntu-based image and an Alpine-based image on the same Linux host because they only ship user-space libraries/binaries; they all still use the host kernel. You cannot, however, run a Windows container on a Linux kernel (or vice versa) without a compatibility layer/VM underneath.

**Q4: Why is "monolithic" written near the VM box on the diagram?**
A: It's drawing the parallel that monolithic apps and VM-based deployments both tend to bundle everything together and scale/update as one big unit — in contrast to containers, which encourage smaller, independently deployable services (microservices).

---

## 2. Where Do We Install Docker?

**What the diagram shows:** "where we need to install docker" → branches to "VM" and "web server."

### Core idea
Docker (the Docker Engine/daemon) is installed **on a host machine** — this host is typically a VM (cloud instance like an EC2/Azure VM) or a bare-metal server. Docker itself is not installed *inside* a container; it's the engine that creates and runs containers. The "web server" box represents the actual workload (Apache/Nginx) that ends up running as a container on top of that Docker-enabled VM.

### Cross Q&A
**Q1: Do you install Docker on the container or on the VM?**
A: On the VM (or physical host). Docker Engine runs on the host OS and then creates containers on top of it.

**Q2: Can you run Docker without any VM at all?**
A: Yes — Docker can run directly on bare-metal Linux servers, not just VMs. Cloud VMs are just a common, convenient host.

---

## 3. Deployment Strategies

**What the diagram shows:** A "Deployment" box listing: **blue green, canary, rolling update, recreate**.

### Core idea — each strategy explained
| Strategy | How it works | Downtime | Risk |
|---|---|---|---|
| **Recreate** | Stop all old version instances, then start all new version instances. | Yes (brief gap) | Simple but users see an outage |
| **Rolling Update** | Gradually replace old instances with new ones, a few at a time, until all are updated. | No (if done right) | Two versions run simultaneously for a while |
| **Blue-Green** | Run two full identical environments ("blue" = current, "green" = new). Switch traffic all at once from blue to green once green is verified. | No | Instant rollback (just switch back), but needs double the infrastructure |
| **Canary** | Release the new version to a small subset of users/traffic first, monitor, then gradually increase the percentage until 100%. | No | Lowest risk — problems affect only a small % of users before full rollout |

### Cross Q&A
**Q1: Which strategy needs double the infrastructure at deployment time?**
A: Blue-Green — you keep both the old (blue) and new (green) environments fully running simultaneously so you can instantly cut over.

**Q2: Which strategy is best if you want to test a new version on real traffic but limit the blast radius of a bug?**
A: Canary — you expose the new version to a small percentage of traffic first and expand gradually.

**Q3: Why does "recreate" cause downtime but rolling update doesn't?**
A: Recreate terminates all old instances before starting new ones, so there's a window with zero running instances. Rolling update always keeps some instances (old or new) serving traffic throughout the process.

**Q4: In Kubernetes terms, which of these are native Deployment strategies vs which need extra tooling?**
A: `Recreate` and `RollingUpdate` are built-in Kubernetes Deployment strategy types. Blue-Green and Canary aren't native Deployment strategies — they're typically implemented using Services/Ingress traffic-shifting, or tools like Argo Rollouts / Flagger / service meshes.

---

## 4. Docker: Host Port vs Container Port

**What the diagram shows:** A "docker" box: "host port and container port" and "shared os and kernel."

### Core idea
`docker run -p <HOST_PORT>:<CONTAINER_PORT> <image>`
- **Host port** — the port opened on the actual machine (VM/laptop/server); this is what you (or the outside world) connect to.
- **Container port** — the port the application *inside* the container is actually listening on (e.g., Nginx listens on 80 by default).
- Traffic flow: **outside world → host port → Docker's internal NAT/bridge → container port → app.**

### Example
```bash
docker run -d -p 8080:80 nginx
```
- `8080` = host port you open in your browser (`http://localhost:8080`)
- `80` = the port Nginx is listening on inside the container

### Cross Q&A
**Q1: In `-p 8080:80`, which number is the host port and which is the container port?**
A: `8080` is the host port (left side), `80` is the container port (right side). The syntax is always `HOST:CONTAINER`.

**Q2: If you change the host port but keep the container port the same, does the application inside the container need to change?**
A: No. The app inside the container still listens on the same port (e.g., 80); only the external mapping changes. This is exactly why you can run Nginx on host port 8080 *and* Apache on host port 8081 while both apps individually still think they're using port 80.

**Q3: What happens if you forget the `-p` flag entirely?**
A: The container still runs and the app still listens on its internal port, but nothing outside the container (or outside the Docker host's internal network) can reach it — there's no port published to the host.

---

## 5. Base Image vs Main (Application) Image vs Container — and Why Latency Happens

**What the diagram shows:** "base image – always lightweight, busybox/alpine" → "main image – build image will take more time" → "container – slow response, latency."

### Core idea
- **Base image**: A minimal starting image (e.g., `alpine`, `busybox`, `scratch`) with just enough OS tooling to build/run on top of. Small size = faster pulls, smaller attack surface.
- **Main (application) image**: Built `FROM` the base image, layering in your app code, dependencies, and runtime. Larger and more complex images take longer to build (and to pull/start).
- **Container**: A running instance of the main image. If the image is bloated (heavy base, unnecessary layers, large dependencies), the container can be slow to pull, start, and respond — this shows up as **latency**, especially in ephemeral/CI environments where a fresh image is pulled every time.

### Why this matters for latency
- A large image means more layers to download over the network before the container can even start — this directly adds to cold-start latency, which matters a lot for CI runners and serverless-style container platforms (like Azure Container Apps Jobs) that spin containers up on demand.

### Cross Q&A
**Q1: Why start `FROM alpine` instead of `FROM ubuntu` for a base image?**
A: Alpine (and busybox) are drastically smaller (a few MB vs. hundreds of MB), which reduces build time, image pull time, storage cost, and attack surface — at the cost of using musl libc instead of glibc, which occasionally causes compatibility issues.

**Q2: If a "main image" build takes a long time, does that affect every container run, or just the first build?**
A: Build time itself is a one-time cost per image version (cached in layers afterward). But a *large resulting image* affects every subsequent `docker pull`/container start, which is the recurring latency cost — this is the distinction the diagram is drawing between "build takes time" (main image) and "slow response/latency" (container/runtime).

**Q3: How would you reduce the "slow response, latency" problem shown for the container box?**
A: Use multi-stage builds (compile in a heavier builder image, copy only the final artifact into a slim runtime image), minimize layers, avoid unnecessary packages, and cache dependency-install layers separately from source-code layers.

---

## 6. CI/CD Runners: CodeBuild Runner (Ephemeral) & Container App Job Runner

**What the diagram shows:** "codebuild runner – ephemeral" → "work for this runner is build the job and once build is done terminate."

### Core idea
- A **CodeBuild runner** (AWS CodeBuild) is used here as the **build** step of the pipeline.
- **Ephemeral** means the runner/compute environment is created fresh for each job and destroyed immediately after — no persistent state, no leftover files or credentials between builds. This is a security and consistency best practice: every build starts from a clean slate.
- Its single job: build the artifact/image, then terminate.

### Cross Q&A
**Q1: What does "ephemeral" mean in the context of a CI runner, and why is it preferred over a long-lived runner?**
A: Ephemeral = spun up per job and torn down right after. It's preferred because it prevents build-to-build contamination (cached secrets, leftover files, dependency drift) and reduces the attack surface — there's no persistent runner for an attacker to compromise and reuse.

**Q2: What's the one job of the CodeBuild runner in this pipeline?**
A: Build the job (compile/package/build the Docker image) — then terminate. It doesn't handle deployment.

---

## 7. Two-Runner, Two-Cloud CI/CD Pipeline (AWS → Azure)

**What the diagram shows:** "we are using 2 runners" — AWS **codebuild runner (build)** → Azure **container app job runner (deploy)**. Also: "managed runner → security concern: shared runners." Plus "Anil – ADCB" and "anis – FAB" each pointing to their own "managed runner."

### Core idea
- This is a **cross-cloud CI/CD pipeline**: the build stage happens in **AWS CodeBuild**, and the deploy stage happens via an **Azure Container App Job runner**.
- **Managed runners** (shared runners provided by a CI platform, e.g., GitHub-hosted runners) are convenient but raise a **security concern**: because they're shared across many users/organizations, there's a theoretical risk of cross-tenant data leakage or noisy-neighbor issues, and you have less control over the underlying environment/compliance posture.
- The diagram shows two people/organizations (**Anil at ADCB**, **anis at FAB** — presumably two different banks) each with their own dedicated managed runner rather than a single shared one — an example of isolating runners per-tenant/org to reduce that shared-runner security concern.
- **Internet access** annotation: CodeBuild runner has internet access (needs it to pull dependencies/push images), while the Container App Job runner is marked "no internet access" — meaning the deploy target sits in a locked-down network (private VNet) that reaches the platform via private endpoints rather than the public internet.

### Cross Q&A
**Q1: Why is build done on AWS but deploy done on Azure in this setup?**
A: Organizations often standardize CI build tooling (CodeBuild) separately from the cloud where the actual application runtime lives (Azure Container Apps). It shows a real hybrid/multi-cloud pipeline where the artifact is built in one cloud and shipped/deployed to workloads hosted in another.

**Q2: What's the security concern with "shared/managed runners"?**
A: Multiple teams or even multiple organizations can execute jobs on the same underlying shared compute pool. This raises concerns about secrets exposure, cross-job contamination, and compliance/isolation — especially sensitive for regulated entities like banks (ADCB, FAB in the notes).

**Q3: Why would the deploy-side runner have "no internet access" while the build-side runner does?**
A: The build runner typically needs outbound internet to fetch dependencies (npm/pip packages, base images) and push the built image to a registry. The deploy runner, on the other hand, often operates inside a private network (VNet) for security — it pulls the already-built image from a private registry/endpoint and doesn't need general internet egress, reducing exposure to exfiltration or supply-chain risk.

**Q4: What does having per-organization dedicated managed runners (Anil/ADCB, anis/FAB) achieve compared to one shared pool?**
A: It provides tenant isolation — each organization's jobs run on runners dedicated to them, avoiding the shared-runner security concern noted above, at the cost of extra infrastructure to manage.

---

## 8. Azure Networking Building Blocks: VNet, Subnets, Route Table, Managed Identity

**What the diagram shows:** "vnet – subnets, route table" and "managed identity" as standalone boxes near the runner architecture.

### Core idea
- **VNet (Virtual Network)**: Azure's isolated private network space for your resources.
- **Subnets**: Subdivisions of the VNet used to segment resources (e.g., one subnet for runners, another for app services) for security and routing control.
- **Route table**: Custom rules that control how network traffic is directed out of a subnet (e.g., forcing traffic through a firewall/NAT gateway instead of straight to the internet) — this is exactly what would enforce the "no internet access" restriction mentioned for the deploy runner.
- **Managed Identity**: An Azure feature that lets a resource (like the Container App Job runner) authenticate to other Azure services (e.g., a container registry, key vault) **without storing credentials/secrets** — Azure AD issues tokens automatically to the resource's identity.

### Cross Q&A
**Q1: How would a route table be used to enforce "no internet access" for the deploy runner shown in section 7?**
A: A custom route table attached to the runner's subnet can force all outbound traffic to a specific next hop (e.g., a firewall appliance or "no route"/blackhole) instead of the default internet route, effectively cutting off general internet egress while still allowing traffic to specific private endpoints.

**Q2: Why use a managed identity instead of storing a username/password or API key for the runner to pull images?**
A: Managed identity eliminates the need to store and rotate long-lived secrets in code or config — Azure automatically handles authentication tokens tied to the resource's identity, reducing the risk of credential leakage.

---

## 9. Dockerfile Instructions: ADD vs COPY, CMD vs ENTRYPOINT

**What the diagram shows:** A "Docker file" box with `CMD ["echo","Hello World"]` and a note "you can change the instruction during runtime"; a "Dockeriel" (Dockerfile) box with `ENTRYPOINT ["echo","Hello"]` and note "fix instruction."

### CMD vs ENTRYPOINT
| Instruction | Purpose | Overridable at `docker run`? |
|---|---|---|
| `CMD` | Provides the **default** command/arguments for the container. | Yes — anything you pass after the image name on `docker run` overrides `CMD` entirely. |
| `ENTRYPOINT` | Defines the **fixed** executable that always runs. | No (by default) — arguments passed at runtime are appended to the ENTRYPOINT rather than replacing it. |

Example:
```dockerfile
CMD ["echo", "Hello World"]
```
```bash
docker run myimage echo "Hello guys"
# Overrides CMD entirely -> prints: Hello guys
```
```dockerfile
ENTRYPOINT ["echo", "Hello"]
```
```bash
docker run myimage "guys"
# ENTRYPOINT is fixed, argument is appended -> prints: Hello guys
```

### ADD vs COPY
From the research notes captured on the board:
- **`COPY`** — Straightforward: copies files/directories from the build context into the image. This is the **recommended default** for most cases because its behavior is simple and predictable.
- **`ADD`** — Does everything `COPY` does, plus two extra features:
  - **Auto-extraction**: If the source is a recognized archive (tar, gzip, bzip2, xz) on the *local* build context, `ADD` automatically unpacks it into the destination.
  - **Remote URL fetching**: `ADD` can take a URL as the source and download it directly into the image.
- Because `ADD`'s "magic" behavior (auto-extract, remote fetch) can produce **unpredictable results** and bloats the build with things like unnecessary downloaded files, `COPY` is preferred unless you specifically need one of `ADD`'s extra features.

### Cross Q&A
**Q1: If a Dockerfile has `CMD ["echo","Hello World"]` and you run `docker run myimage echo "Hi"`, what gets printed?**
A: `Hi` — any command specified at `docker run` completely replaces `CMD`.

**Q2: If a Dockerfile has `ENTRYPOINT ["echo","Hello"]` and you run `docker run myimage "guys"`, what gets printed?**
A: `Hello guys` — arguments given at runtime are appended to (not replacing) the ENTRYPOINT.

**Q3: Why might you combine ENTRYPOINT and CMD together in the same Dockerfile?**
A: ENTRYPOINT sets the fixed program to run, and CMD supplies default arguments to it that a user *can* override at runtime — e.g. `ENTRYPOINT ["echo"]` + `CMD ["Hello World"]` lets `docker run myimage` print "Hello World" by default, while `docker run myimage "Hi"` overrides just the argument.

**Q4: When should you use `ADD` instead of `COPY`?**
A: Only when you specifically need its extra behavior — auto-extracting a local tarball into the image, or fetching a remote file by URL. For plain file copying, `COPY` is recommended because it's simpler and its behavior can't surprise you.

**Q5: Why is "unpredictable behavior" cited as a downside of `ADD`?**
A: Because `ADD` silently changes behavior based on the source type (plain file vs. archive vs. URL) — someone reading the Dockerfile might not immediately realize a "copy" is actually extracting an archive or fetching from the internet, which can introduce build inconsistency or unexpected content.

---

## 10. Docker Networking Drivers

**What the diagram/notes show:** bridge, host, overlay, macvlan/ipvlan, none.

### Core idea — each driver explained
| Driver | Behavior |
|---|---|
| **bridge** (default) | Creates a private internal network on a single host; containers get an internal IP and can talk to each other, but need explicit port forwarding (`-p`) to be reached from outside. |
| **host** | Removes network isolation between the container and host — the container binds directly to the host's network stack/ports, giving higher performance (no NAT) but less isolation. |
| **overlay** | Connects multiple Docker daemons/hosts together; the foundation for multi-host networking such as Docker Swarm. |
| **macvlan / ipvlan** | Assigns the container a MAC or IP address directly on the physical network, making it appear as its own physical device on the LAN. |
| **none** | Disables networking entirely for the container — complete isolation. |

Best practice: while Docker auto-creates a default bridge network, creating a **user-defined bridge network** is recommended for production because it gives containers automatic DNS resolution by container name (rather than only by IP) and better isolation between unrelated groups of containers.

### Cross Q&A
**Q1: Why does the default `bridge` network require the `-p` flag to be reachable from outside, while `host` mode doesn't?**
A: `bridge` mode puts the container on its own isolated internal network with a private IP, so external traffic must be explicitly forwarded (published) to a host port via `-p`. `host` mode removes that isolation entirely — the container shares the host's network stack directly, so whatever port the app listens on is immediately exposed on the host with no mapping needed.

**Q2: Which driver would you use to link containers across multiple physical Docker hosts, e.g., in a Swarm cluster?**
A: `overlay` — it's specifically designed for multi-host container-to-container communication.

**Q3: Why would you choose a user-defined bridge network over the default bridge network in production?**
A: User-defined bridge networks provide automatic DNS-based service discovery (containers can reach each other by container name), and better network segmentation, whereas the default bridge only supports linking by IP (or legacy `--link`).

**Q4: What's the tradeoff of using `macvlan`/`ipvlan`?**
A: You get a container that looks like a real device on your physical LAN (its own MAC/IP), which is useful for legacy apps expecting real network presence, but it requires more careful network configuration and typically doesn't work well with cloud provider virtual networks that restrict MAC spoofing.

---

## 11. Web Architecture: WAF → Load Balancer → App → Ingress

**What the diagram shows:** Actor → **WAF (layer 7 security, rules)** [IP: 20.120.49.10] → **LB (App Gateway LB / App LB, TLS termination)** → `business.adcb.com` → **Ingress**.

### Core idea
- **Actor**: The end user/client making a request.
- **WAF (Web Application Firewall)**: Operates at **Layer 7** (application layer), inspecting HTTP(S) traffic against security rules (e.g., blocking SQL injection, XSS, bad bots) before it ever reaches your infrastructure.
- **LB / Application Gateway**: A layer-7-aware load balancer (e.g., Azure Application Gateway) that also performs **TLS termination** — decrypting HTTPS at the gateway so backend services can communicate over plain HTTP internally (simplifying certificate management) — and then distributes traffic across backend targets.
- **`business.adcb.com`**: The public-facing hostname that resolves through this chain.
- **Ingress**: Inside the cluster (Kubernetes), the Ingress resource/controller takes the traffic that reaches the cluster and routes it to the correct internal Service based on hostname/path rules.

### Request flow, end to end
`Actor → WAF (filters malicious traffic) → App Gateway/LB (TLS termination + load balancing) → business.adcb.com (DNS/hostname) → Ingress (in-cluster routing) → Service → Pod`

### Cross Q&A
**Q1: What does "Layer 7" mean, and why does it matter for a WAF?**
A: Layer 7 is the Application layer of the OSI model — the layer where HTTP requests, headers, cookies, and payload content live. A Layer-7-aware WAF can inspect the actual content of requests (not just IP/port info from lower layers), letting it detect attacks like SQL injection or XSS hidden inside the request body.

**Q2: What is "TLS termination" and why do it at the load balancer instead of at each backend pod?**
A: TLS termination means decrypting incoming HTTPS traffic at the load balancer/gateway, so traffic downstream can travel as plain HTTP. Centralizing it at the gateway simplifies certificate management (one place to renew/rotate certs) and offloads the CPU cost of encryption/decryption from every individual backend instance.

**Q3: What's the difference between the Load Balancer/App Gateway shown here and the Ingress box?**
A: The LB/App Gateway operates outside the cluster, handling public entry, WAF rules, and TLS termination. The Ingress resource operates inside the Kubernetes cluster, routing already-arrived traffic to the correct internal Service based on host/path rules. In many Azure setups, the App Gateway can even act as the Ingress Controller itself (Application Gateway Ingress Controller/AGIC), bridging both roles.

---

## 12. Kubernetes: Service → Deployment → Pods → Horizontal Pod Autoscaler (HPA)

**What the diagram shows:** **service** → **deployment** → pods 1, 2, 3, 4 (chained), plus a box: `cpu 60, memory 70, min: 3, max: 10`.

### Core idea
- **Service**: A stable network endpoint (virtual IP/DNS name) that load-balances traffic across a set of Pods, even as individual Pods are replaced.
- **Deployment**: A controller that manages a set of identical Pods (via ReplicaSets), handling rollouts, rolling updates, and self-healing (recreating Pods that die).
- **Pods**: The smallest deployable unit in Kubernetes — one or more containers sharing network/storage, scheduled together on a node.
- **HPA (Horizontal Pod Autoscaler)**: Automatically scales the *number* of Pod replicas up or down based on observed metrics.
  - `cpu 60` → target average CPU utilization of 60% across pods.
  - `memory 70` → target average memory utilization of 70%.
  - `min: 3` → never scale below 3 pods (baseline availability/redundancy).
  - `max: 10` → never scale above 10 pods (cost/resource ceiling).

### How it fits together
`Service (stable entrypoint) → Deployment (manages desired pod count & rollout strategy) → Pods (running app instances) ← HPA watches CPU/memory and adjusts Pod count between 3 and 10`

### Cross Q&A
**Q1: What's the difference between a Service and a Deployment in Kubernetes?**
A: A Deployment manages the actual Pods — creating, updating, and healing them to match a desired state. A Service provides a stable, unchanging way to *reach* those Pods (a fixed IP/DNS name), even as the underlying Pods are replaced/rescheduled and get new IPs.

**Q2: With `min: 3, max: 10`, what happens under very low load?**
A: The HPA will never scale below 3 replicas, regardless of how low CPU/memory usage drops — this floor guarantees baseline redundancy/availability (so you're never down to a single point of failure).

**Q3: With `cpu 60, memory 70` as HPA targets, what triggers a scale-up?**
A: If the average CPU utilization across pods exceeds 60% (or average memory exceeds 70%, depending on which metrics the HPA is configured to watch), Kubernetes will add more Pod replicas (up to the max of 10) to bring the average back down toward the target.

**Q4: Why does Kubernetes scale the *number of Pods* (horizontal) rather than giving each Pod more CPU/memory (vertical) by default with HPA?**
A: Horizontal scaling (adding more replicas) is generally safer and less disruptive — it doesn't require restarting existing Pods and it improves both capacity and redundancy simultaneously. Vertical scaling (resizing a Pod's resource requests/limits) typically requires a Pod restart and is handled by a separate mechanism (Vertical Pod Autoscaler), not the standard HPA.

**Q5: If a Pod crashes, does the HPA restart it?**
A: No — that's the Deployment's (and underlying ReplicaSet's) job, which continuously ensures the desired number of healthy Pods exists. The HPA's only job is deciding *how many* replicas should exist based on metrics; it changes the desired count, and the Deployment/ReplicaSet is what actually creates/restarts Pods to match that count.

---

## Quick Recap Table

| Topic | Key Terms |
|---|---|
| Virtualization | Monolithic, VM, Container, single shared kernel |
| Docker basics | `docker run -p host:container`, image versioning |
| Deployment strategies | Recreate, Rolling Update, Blue-Green, Canary |
| Image layers | Base image (alpine/busybox) → Main image → Container, latency |
| CI/CD runners | Ephemeral CodeBuild runner (build), Container App Job runner (deploy), managed/shared runner risk |
| Azure networking | VNet, Subnet, Route Table, Managed Identity |
| Dockerfile | CMD (overridable) vs ENTRYPOINT (fixed), COPY vs ADD |
| Docker networking | bridge, host, overlay, macvlan/ipvlan, none |
| Edge architecture | WAF (L7), App Gateway/LB (TLS termination), Ingress |
| Kubernetes | Service, Deployment, Pods, HPA (cpu/memory targets, min/max) |
