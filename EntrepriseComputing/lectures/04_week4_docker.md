# Week 4 — Virtualization and Containerization

## Bird's eye view

- The lecture traces the evolution from **traditional bare-metal deployment** through **hardware virtualization (VMs)** to **OS-level containerization**, motivating why each step was necessary.
- **Virtual Machines** (via a hypervisor) give each workload its own guest OS — strong isolation but gigabyte images, minute-scale boot times, and instruction-translation overhead.
- **Containers** virtualize the OS rather than the hardware: they share the host kernel (isolated via Linux **namespaces** + **cgroups**), producing megabyte images that start in seconds with near-native CPU performance.
- **Docker** is the dominant container platform: an open-source engine that packages the Build → Ship → Run workflow into a unified API (daemon + CLI + registry).
- A **Dockerfile** is the source blueprint; `docker build` produces an immutable, layered **image**; `docker run` instantiates a **container** (a running image with a thin writable layer on top).
- **Docker Compose** orchestrates multi-container applications on a single host; **Kubernetes** (K8s) or **Docker Swarm** orchestrate containers at cluster scale.
- Empirical benchmarks show Docker consistently uses less CPU and has tighter performance variance than an equivalent VM workload.

---

## Detailed notes

### 1. The Evolution of Resource Abstraction

#### 1.1. Historical timeline

| Era | Event |
|---|---|
| **1960s** | MIT CTSS — time-sharing, multiprogramming, moving away from batch jobs. |
| **Virtual Memory** | Mapping virtual to physical addresses; processes see contiguous address space — paved the way for hypervisors. |
| **1972** | IBM System/370 — first hardware-assisted virtualization features. |
| **1999** | VMware x86 virtualization; later Intel VT-x (hardware-assisted) eliminated the need for binary translation of protected-mode instructions. |

Early virtualization required **ring deprivileging** and **binary translation** to handle privileged instructions without trapping the kernel.

#### 1.2. The three deployment generations (diagram description)

The slide shows three stacks side by side:
- **Traditional deployment**: multiple apps share one OS on bare hardware. No isolation.
- **Virtualized deployment**: a hypervisor sits between hardware and the OS; each VM has its own guest OS + bins/libs + apps. Strong isolation, heavy overhead.
- **Container deployment**: a container runtime sits on one OS; each container has only its bins/libs + app. Lightweight, shares the host kernel.

---

### 2. Hypervisor Types

| Type | Description | Examples |
|---|---|---|
| **Type 1 (Bare-Metal)** | Runs directly on hardware. No host OS overhead. High performance, high security. | VMware ESXi, Xen |
| **Type 2 (Hosted)** | Runs as software on a host OS. Easier to set up, but has extra OS-layer latency. | VirtualBox, VMware Workstation |

- **Full Virtualization**: uses binary translation to mimic hardware for unmodified guest OS.
- **Paravirtualization**: modifies the guest OS kernel to make **hypercalls** directly to the hypervisor — faster but requires OS cooperation.

---

### 3. OS-Level Virtualization: The Linux Kernel Mechanisms

Containers **do not virtualize hardware**. They virtualize the OS and share the host kernel, maintaining isolated user spaces. Two kernel features make this possible:

#### 3.1. Namespaces (isolation)

A namespace restricts what a process can *see*. Six namespace types:

| Namespace | What it isolates |
|---|---|
| **Mount (mnt)** | Filesystem mount points |
| **UTS** | Hostname and domain name |
| **IPC** | Inter-process communication (message queues, semaphores) |
| **PID** | Process ID tree (container sees its own PID 1) |
| **Network** | Network interfaces, routing tables, ports |
| **User** | UID/GID mappings (container root ≠ host root) |

#### 3.2. cgroups (control groups — resource accounting and limits)

cgroups limit and account for resource usage per process group. Example `/etc/cgconfig.conf`:

```bash
group limitcpu {
    cpu { cpu.shares = 400; }
}
group limitmem {
    memory { memory.limit_in_bytes = 512m; }
}
group limitio {
    blkio { blkio.throttle.read_bps_device = "252:0  2097152"; }
}
```

cgroups control: **CPU shares**, **memory limits**, **disk I/O throttling**, and **network bandwidth**.

---

### 4. VMs vs. Containers: The Architecture Showdown

The slide shows two stacks:
- **VM stack** (bottom to top): Server → Host OS → Hypervisor → Guest OS → Bins/Libs → App. Each VM carries a full guest OS.
- **Container stack** (bottom to top): Server → Host OS → Docker Engine → (shared kernel) → Bins/Libs → App. No guest OS per container.

| Metric | Virtual Machine | Container |
|---|---|---|
| **Image Size** | Gigabytes (full OS) | Megabytes (app + binaries) |
| **Boot Time** | Minutes (BIOS + OS boot) | Seconds (process start) |
| **Performance** | Instruction translation overhead | Near-native (shared kernel) |
| **Portability** | Hypervisor-dependent | Consistent everywhere |

---

### 5. Docker: Architecture, Lifecycle, and Ecosystem

**Platform definition**: Docker is an open platform to **build, ship, and run** distributed applications, eliminating friction between development, QA, and production environments.

The lifecycle maps to three verbs:
- **Build**: Source code + Dockerfile → Image (immutable snapshot). Packages app code, dependencies, and runtime.
- **Ship**: Push image to a Registry (Docker Hub, private registry) for global distribution.
- **Run**: Docker Engine instantiates the image as an isolated, portable container (CPU, RAM, network allocated).

#### 5.1. Docker Ecosystem (3-column architecture diagram)

```
Client (CLI / Remote API)  ──►  Host (Daemon)  ──►  Registry (Hub)
                                    │
                          ┌─────────┴─────────┐
                       Containers          Images  ◄── pulled from Hub
```

- **Docker Daemon** (`dockerd`): background process managing images, containers, networks, and volumes. Listens on a Unix socket or TCP.
- **Docker Client** (`docker` CLI): sends REST API commands to the daemon.
- **Registry** (Docker Hub): central image storage. `docker pull` fetches, `docker push` uploads.

#### 5.2. UnionFS — the layered filesystem

Images are built **layer by layer** using a Union File System (e.g., OverlayFS). Base layers are **read-only**. When a container starts, Docker adds one thin **read/write layer** on top — maximizing disk efficiency through layer sharing.

---

### 6. Image Anatomy

The image layer stack (bottom to top):

```
[ Container Layer ]   ← Read/Write  (per running container, ephemeral)
[ Add Apache      ]   ← Read-Only
[ Add Emacs       ]   ← Read-Only   } Copy-on-Write storage strategy
[ Base Image (Debian) ] ← Read-Only
[ Kernel (Host)   ]   ← Shared
```

- **Image ID**: random hex string.
- **Digest**: content-addressed SHA256 hash — guarantees integrity.
- **Tag**: human-readable pointer, e.g., `nginx:latest`. `:latest` is implicit but not always the newest — always prefer explicit version tags in production.
- **Copy-on-Write (CoW)**: when a container modifies a read-only file, it is first copied up to the writable container layer. The original image layer is untouched.

---

### 7. The Dockerfile

Dockerfile = source code for an image. Image = compiled binary.

```dockerfile
FROM debian:stretch          # Base image (first instruction, mandatory)
RUN apt-get install python   # Execute during build; commits a new layer
COPY . /app                  # Copy files from build context into image
WORKDIR /app                 # Set working directory for subsequent instructions
CMD ["python", "app.py"]     # Default command at runtime (becomes PID 1)
```

Full instruction reference:

| Instruction | Purpose |
|---|---|
| `FROM` | Sets the base image. Every Dockerfile must start here. |
| `RUN` | Executes a shell command during build; each `RUN` creates a new layer. |
| `COPY` | Copies files/dirs from the build context (host) into the image. |
| `ADD` | Like `COPY` but also unpacks local tar archives and supports URLs. Prefer `COPY` unless you need these extras. |
| `WORKDIR` | Sets the working directory for `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`. |
| `EXPOSE` | Documents which port the container listens on (informational; does not publish). |
| `CMD` | Default command when the container starts. Can be overridden at `docker run`. |
| `ENTRYPOINT` | Fixed command; `CMD` becomes its arguments. Use when the container has a single purpose. |
| `ENV` | Sets environment variables available at build time and runtime. |
| `VOLUME` | Declares a mount point; Docker creates an anonymous volume automatically. |

#### 7.1. A complete realistic Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV FLASK_ENV=production
EXPOSE 5000

CMD ["python", "app.py"]
```

---

### 8. Advanced Build Mechanics

#### 8.1. Build cache

Docker checks each instruction against a cached layer. If the instruction and its inputs are unchanged, the cached layer is reused. **If any layer is invalidated, all subsequent layers are rebuilt.**

Ordering rule: **put stable, slow instructions at the top** (e.g., `RUN apt-get install`), **and frequently-changing instructions at the bottom** (e.g., `COPY . .`).

```dockerfile
# Good ordering — dependencies cached unless requirements.txt changes
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .                     # Only this layer rebuilds on source changes
```

#### 8.2. Multi-stage builds

Reduces final image size by using a heavyweight builder stage and copying only the compiled artifact into a minimal production image.

```dockerfile
# Stage 1: Builder
FROM golang:1.21 AS builder
WORKDIR /src
COPY . .
RUN go build -o /app/server .

# Stage 2: Production (minimal image)
FROM alpine:3.19
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

The production image contains only the binary — compilers, source code, and build tools are **left behind** in the builder stage.

---

### 9. Container Lifecycle

The lifecycle state machine (from the slides):

```
Image ──docker create──► Created ──docker start──► Running
                                                      │         │
                                               docker pause   docker stop (SIGTERM + 10s grace)
                                                      │         │
                                                   Paused    Exited ──docker rm──► Destroyed
                                             docker unpause ──►/        │
                                                              Filesystem persists in Exited state
```

- **Running**: PID 1 (the CMD/ENTRYPOINT process) is active.
- **Paused**: processes frozen via cgroup freezer; filesystem intact.
- **Exited**: PID 1 terminated; writable container layer **persists on disk** until `docker rm`.
- **Destroyed**: `docker rm` drops the writable layer permanently.

---

### 10. Key Docker Commands

#### Container management

```bash
# Run a container (create + start in one step)
docker run -d -it --rm --name web-server -p 80:80 nginx
#   -d         detached (background), returns container ID
#   -it        interactive + allocate pseudo-TTY
#   --rm       auto-remove filesystem on exit
#   --name     assign a human-readable name
#   -p 80:80   publish host_port:container_port

docker ps                    # List running containers
docker ps -a                 # List all containers (including stopped)
docker stop <id|name>        # Send SIGTERM, then SIGKILL after 10s grace
docker rm <id|name>          # Remove stopped container
docker exec -it <id> bash    # Open a shell in a running container
docker logs <id>             # Fetch container stdout/stderr
docker inspect <id>          # Low-level JSON metadata
```

#### Image management

```bash
docker build -t myapp:1.0 .  # Build image from Dockerfile in current dir
docker images                # List local images
docker pull nginx:latest     # Download image from registry
docker push myrepo/myapp:1.0 # Upload image to registry
docker rmi myapp:1.0         # Remove local image
```

---

### 11. Networking

By default containers are attached to a private **bridge network** (`docker0`, subnet `172.17.x.x`). They are isolated from the host and from each other unless configured otherwise.

#### Network diagram description

External traffic arrives at the host boundary. Port mapping (`-p host_port:container_port`) punches a hole through NAT to reach a specific container. Containers on the same user-defined bridge network can resolve each other by **container name** via Docker's embedded DNS.

#### Network drivers

| Driver | Description |
|---|---|
| **Bridge** | Default. Private virtual network on the host. Containers communicate via container IPs or names (on user-defined networks). |
| **Host** | Container shares the host's network namespace directly — no NAT, no isolation. Maximum performance, zero port mapping. |
| **Overlay** | Spans multiple Docker hosts (used by Swarm and Kubernetes). Enables container-to-container communication across nodes. |
| **None** | Disables all networking for the container. |

```bash
# Create a user-defined bridge (enables DNS by container name)
docker network create my-net

# Run containers on the custom network
docker run -d --name db --network my-net postgres:15
docker run -d --name app --network my-net -e DB_HOST=db myapp:1.0
```

---

### 12. Volumes and Persistence

Container filesystems are **ephemeral** — data in the writable layer is lost when the container is removed. The two mechanisms to externalise data:

#### Volume diagram description

Two pipes connect the container box to the host system. One pipe (Bind Mount) goes to a specific host directory (e.g., `/home/user/code`). The other pipe (Named Volume) goes to Docker's managed area (`/var/lib/docker/volumes/`) with its own independent lifecycle.

| Type | Managed by | Best for |
|---|---|---|
| **Named Volume** | Docker engine | Production databases, stateful services. Survives container removal. |
| **Bind Mount** | Host OS | Development: live-reload source code into container. |

```bash
# Named volume (Docker manages the path)
docker run -d -v my-db-data:/var/lib/postgresql/data postgres:15

# Bind mount (host path explicitly specified)
docker run -d -v /home/user/code:/app myapp:1.0

# In a Dockerfile, declare a volume mount point
VOLUME ["/var/lib/postgresql/data"]
```

---

### 13. Docker Compose

Docker Compose defines and runs **multi-container applications** as Infrastructure as Code, using a single YAML file.

```yaml
# docker-compose.yml
version: "3.9"
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

```bash
docker-compose up -d         # Start all services in background
docker-compose down          # Stop and remove containers (volumes preserved)
docker-compose down -v       # Also remove named volumes
docker-compose logs -f       # Follow logs from all services
docker-compose ps            # List services and their status
```

Compose creates a **dedicated bridge network** for the application; services resolve each other by service name (e.g., `db` is reachable at `db:5432` from `web`).

---

### 14. Security

#### 14.1. The shared kernel vulnerability

Unlike VMs — which are trapped by a hypervisor — containers share the host kernel. **A kernel exploit in a container can compromise the entire host.** Threat vectors:

- **Container breakout** (e.g., CVE-2019-5736 in runc): malicious process escapes to host root.
- **ARP spoofing** on bridged networks.
- **DoS via resource exhaustion** (mitigated by cgroups limits).
- **Symlink attacks** via malicious volume mounting.

**Golden rule: never run as root. Use `docker run -u USER` or add `USER` to the Dockerfile.**

#### 14.2. Defense in depth

| Mechanism | What it does |
|---|---|
| **Root Privilege Restriction** | Run container as non-root user. Limits blast radius. |
| **Capabilities (`--cap-drop=all`)** | Drop dangerous Linux capabilities; add back only what is needed. |
| **Seccomp profiles** | Syscall whitelisting — blocks dangerous calls like `ptrace` and admin calls. |
| **User Namespaces** | Maps container root (UID 0) to host unprivileged user (UID 65534). Even if container root is compromised, host sees a nobody user. |
| **Image Signing (Docker Content Trust)** | Cryptographic verification that images are from a trusted publisher. |
| **Network Isolation** | Use dedicated user-defined bridge networks; avoid the default bridge. |

```bash
# Run with dropped capabilities and a non-root user
docker run --cap-drop=all --user 1001 myapp:1.0
```

---

### 15. Docker Ecosystem Tools

#### 15.1. Docker Machine

Provisions Docker hosts on local VMs (VirtualBox), public cloud (AWS, Azure, GCP), or private cloud. Used to manage remote Docker daemons.

```bash
docker-machine create --driver virtualbox my-host
```

#### 15.2. Orchestration: Docker Swarm vs. Kubernetes

At scale, managing thousands of ephemeral containers manually is impossible. Containers die, IPs change, and scaling requires automation.

| Feature | Docker Swarm | Kubernetes (K8s) |
|---|---|---|
| **Origin** | Native Docker, embedded | Google (CNCF standard) |
| **Complexity** | Simple API, easy setup | Complex, steep learning curve |
| **Security** | Secure by default (TLS) | Requires explicit configuration |
| **Adoption** | Declining | Industry standard |

**Kubernetes solves**: desired state reconciliation, automatic service discovery (DNS), auto-healing (restarts failed containers), and load balancing across replicas.

##### Kubernetes Control Plane components (architecture diagram description)

The control plane is a set of services that communicate through `kube-apiserver`, which is the single front-end for all REST calls:

- **kube-apiserver**: handles all REST API calls; the front-end for the control plane.
- **etcd**: highly-available key-value store that holds all cluster state — the brain.
- **kube-scheduler**: assigns new Pods to Nodes based on resource availability — the dispatcher.
- **kube-controller-manager**: runs control loops to maintain desired state (e.g., replica counts) — the regulator.
- **cloud-controller-manager**: integrates with cloud provider APIs (load balancers, storage).

##### Kubernetes Node-level components

- **Kubelet**: agent on every node; ensures containers in Pods are running and healthy.
- **Kube-proxy**: maintains network rules (iptables/IPVS) for Pod-to-Pod and Service communication.
- **Pod**: the atomic scheduling unit — one or more containers sharing network and storage.

##### Pod lifecycle states

```
Pending → Running → Succeeded (all containers terminated successfully)
                 ↘ Failed    (one or more containers terminated with error)
Unknown          (Kubelet stopped reporting to API server)
```

---

### 16. Real-World Microservices Case Study: Voting App on Docker Swarm

The "Voting App" demonstrates a polyglot microservices architecture on Docker Swarm with two isolated overlay networks:

**Frontend Network (overlay)**:
- Python Web App (Vote) — 2 replicas — writes votes to Redis Queue.
- Redis Queue — message broker between frontend and worker.

**Backend Network (overlay)**:
- Worker (.NET Core) — reads from Redis, writes to PostgreSQL.
- PostgreSQL (DB) — persistent storage.
- Node.js Web App (Result) — reads from PostgreSQL, displays results.

The Worker service bridges both networks. Each service runs in its own container, using a different language — demonstrating Docker's language-agnostic portability.

---

### 17. Performance Benchmarks

**Setup**: HP DL380 Gen8, dual Intel Xeon E5-2680 v2 (10 cores, 2.80GHz each), 128 GB RAM, Ubuntu 20.10.

| Test | Method |
|---|---|
| Web Server (Test A) | Apache 2.4.0, 20,000 HTTP requests, 1,000 concurrent connections, with/without KeepAlive |
| File Server (Test B) | Samba 4.13.17, transferring a 3.1 GB file over Ethernet |

**Results**: Docker demonstrated lower median CPU usage and **tighter variance** (smaller interquartile range) than VMs in both tests. VM overhead is attributed to instruction translation and context switching at the hypervisor layer.

**Conclusion**: Docker achieves near-native performance because there is no instruction translation — the container processes run directly on the host kernel.

---

### 18. The Open Container Initiative (OCI)

OCI is the industry standard that decouples the container format from any single vendor:

- **Runtime-spec** (`runc`): defines how to launch a container. Docker uses `runc` as its low-level runtime.
- **Image-spec**: defines the standardized image format. Any OCI-compliant runtime can run an OCI image.

This means Docker images run on Kubernetes, Podman, containerd, and other OCI-compliant runtimes without modification.

---

## Key terms

- **Hypervisor** — software that abstracts hardware to run multiple VMs; Type 1 (bare-metal) or Type 2 (hosted).
- **Namespace** — Linux kernel feature that isolates a process group's view of system resources (PID, network, mount, UTS, IPC, user).
- **cgroups** — Linux kernel feature that limits and accounts for resource usage (CPU, memory, disk I/O) per process group.
- **Docker Daemon** (`dockerd`) — background service that manages images, containers, networks, and volumes.
- **Image** — immutable, layered, read-only artifact built from a Dockerfile. The "compiled binary."
- **Container** — a running instance of an image with a thin writable layer added on top. Ephemeral by default.
- **Dockerfile** — text file of instructions that define how to build an image. The "source code."
- **Layer** — a single immutable filesystem change recorded during a `RUN`, `COPY`, or `ADD` instruction.
- **UnionFS / OverlayFS** — filesystem that merges multiple read-only layers with one writable layer into a single coherent view.
- **Copy-on-Write (CoW)** — strategy where a file is copied to the writable layer only when it is modified; original layers are untouched.
- **Build cache** — Docker reuses a cached layer if the instruction and its inputs have not changed.
- **Multi-stage build** — Dockerfile pattern using multiple `FROM` statements to produce a minimal production image by discarding build-time tools.
- **Named volume** — Docker-managed persistent storage with an independent lifecycle from any container.
- **Bind mount** — host directory path mapped into the container; changes are immediately visible on both sides.
- **Bridge network** — default Docker network driver; containers on the same bridge communicate via private IPs.
- **Overlay network** — Docker network that spans multiple hosts; used by Swarm and Kubernetes.
- **Docker Compose** — tool for defining and running multi-container applications from a single YAML file.
- **Docker Hub** — default public registry for Docker images.
- **OCI (Open Container Initiative)** — industry standard for container runtime-spec and image-spec; ensures interoperability.
- **Pod** — Kubernetes' atomic scheduling unit; one or more tightly coupled containers sharing network and storage.
- **Kubernetes** — CNCF standard container orchestration platform; manages desired state, service discovery, scaling, and self-healing at cluster scale.

---

## Exam targets

1. **Draw the containers vs. VMs architecture** — show the stack layers for each and identify where the guest OS is (or is not) present. Explain the performance implications.
2. **Explain the two Linux kernel mechanisms** that make containers possible: namespaces (isolation of what a process *sees*) and cgroups (limits on what a process *uses*). Name the six namespace types.
3. **Explain the Docker architecture** — the three components (client, daemon, registry) and their interactions. Describe UnionFS and why image layers are read-only.
4. **Write a minimal Dockerfile** for a Python or Node.js web app, using correct instruction order to maximize build cache hits. Explain what each instruction does.
5. **Explain the container lifecycle** — the states (Created, Running, Paused, Exited, Destroyed) and the commands that trigger transitions.
6. **Explain `docker run` flags**: `-d`, `-it`, `--rm`, `--name`, `-p host:container`, `-v`.
7. **Compare named volumes vs. bind mounts** — when to use each, and why container filesystems are ephemeral.
8. **Explain Docker networking**: bridge (default, private), host (shared kernel network), overlay (multi-host). Describe how port mapping works (NAT).
9. **Explain multi-stage builds** — why they reduce image size, and how `COPY --from=<stage>` works.
10. **Explain the security trade-off**: containers share the kernel (less isolated than VMs). Describe three defense mechanisms: cap-drop, seccomp, user namespaces.
11. **Compare Docker Swarm vs. Kubernetes** — simplicity vs. power, use cases for each.

---

## Pitfalls

- **Containers are NOT VMs.** They do not have their own kernel. A Linux container requires a Linux kernel — on macOS/Windows, Docker runs a lightweight Linux VM to host the engine.
- **`:latest` is not always the latest.** It is just a default tag that maintainers may not update. Always use explicit version tags in production (e.g., `python:3.11-slim`).
- **`EXPOSE` does not publish a port.** It is documentation only. Publishing requires `-p` at `docker run` time (or `ports:` in Compose).
- **CMD vs. ENTRYPOINT confusion.** `CMD` is the default command and can be overridden by anything appended to `docker run`. `ENTRYPOINT` is fixed; `CMD` becomes its arguments. If both are set in exec form, they combine.
- **Build cache invalidation is linear.** Changing any layer invalidates **all layers below it** in the Dockerfile. `COPY . .` must come after dependency installation, not before.
- **Stopped containers still consume disk.** `docker ps` only shows running containers. Use `docker ps -a` to see stopped ones. Run `docker rm` or use `--rm` flag to clean up.
- **`docker stop` vs. `docker kill`**: `stop` sends SIGTERM and waits 10 seconds (grace period for cleanup) before sending SIGKILL. `kill` sends SIGKILL immediately — no cleanup.
- **Named volumes vs. bind mounts**: bind mounts expose the host filesystem to the container — a security risk in production. Named volumes are preferred for stateful workloads.
- **Running as root is dangerous.** A root process in the container that achieves kernel exploit can become root on the host. Always add `USER` to the Dockerfile or use `--user` at runtime.
- **The default bridge network does not support DNS by container name.** You must create a user-defined bridge network for container-name-based DNS resolution to work.
- **`docker-compose down` does not remove named volumes by default.** You must add `-v` explicitly — otherwise data persists, which is usually the desired behavior but can cause confusion.
