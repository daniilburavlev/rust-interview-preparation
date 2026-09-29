# Docker and Containers: Senior Interview Questions

Container internals, image building, security and running containers in production, with notes specific to Rust services.

## Contents

1. [Fundamentals](#1-fundamentals)
2. [Images and Dockerfiles](#2-images-and-dockerfiles)
3. [Rust-specific images](#3-rust-specific-images)
4. [Networking and storage](#4-networking-and-storage)
5. [Security](#5-security)
6. [Running in production](#6-running-in-production)
7. [Kubernetes basics](#7-kubernetes-basics)

---

## 1. Fundamentals

### 1.1 Container vs virtual machine?

A VM virtualizes hardware and runs its own kernel via a hypervisor. A container is an isolated process sharing the host kernel, isolated with Linux namespaces and limited with cgroups. Containers start in milliseconds and are lighter, but provide weaker isolation (a kernel exploit affects all containers). Sandboxes like gVisor, Kata Containers and Firecracker narrow that gap.

### 1.2 Which Linux features make containers work?

- **Namespaces**: isolate PID, network, mount, UTS (hostname), IPC, user and cgroup views.
- **cgroups**: limit and account CPU, memory, I/O, PIDs.
- **Union/overlay filesystems**: layered images (overlay2).
- **Capabilities, seccomp, AppArmor/SELinux**: restrict what the process can do.

### 1.3 What are Docker Engine, containerd and runc? What is OCI?

The Docker CLI talks to `dockerd`, which delegates to `containerd` (container lifecycle, image management), which uses `runc` (low-level OCI runtime creating namespaces/cgroups). OCI (Open Container Initiative) standardizes image format and runtime spec, so images built by Docker run on containerd, CRI-O, Podman, and Kubernetes.

### 1.4 Image vs container vs layer?

An image is an immutable stack of read-only layers plus config (entrypoint, env). A container is a running (or stopped) instance with a thin writable layer on top. Layers are content-addressed and shared between images, saving disk and download time.

---

## 2. Images and Dockerfiles

### 2.1 `CMD` vs `ENTRYPOINT`? Shell vs exec form?

`ENTRYPOINT` defines the executable; `CMD` provides default arguments (or the command if there's no entrypoint), overridable at `docker run`. Use the exec form (`["/app", "--flag"]`): the shell form wraps the command in `/bin/sh -c`, so your process isn't PID 1 and doesn't receive SIGTERM directly.

### 2.2 How does layer caching work and how do you order instructions?

Each instruction creates a layer; if an instruction and its inputs haven't changed, the cached layer is reused, and a change invalidates all subsequent layers. Order from least to most frequently changing: base image, system packages, dependency manifests and dependency install, then source code.

### 2.3 What are multi-stage builds?

Multiple `FROM` stages in one Dockerfile: build in a full toolchain image, then `COPY --from=builder` only the artifact into a minimal runtime image. Results in smaller, more secure images without build tools.

### 2.4 `COPY` vs `ADD`?

`COPY` copies local files. `ADD` also extracts local tar archives and can fetch URLs, which is surprising and less secure. Prefer `COPY`.

### 2.5 What is BuildKit and which features matter?

The modern build engine: parallel stage execution, cache mounts (`RUN --mount=type=cache,target=...`) to persist package caches between builds, secret mounts (`--mount=type=secret`) so credentials never end up in layers, SSH forwarding, and remote cache export/import (`--cache-to/--cache-from`) for CI.

### 2.6 How do you build multi-architecture images?

`docker buildx build --platform linux/amd64,linux/arm64 --push`, producing a manifest list. Use native builders or cross-compilation instead of QEMU emulation for speed (emulated Rust builds are very slow). Relevant for AWS Graviton (arm64).

### 2.7 How do you keep images small?

Multi-stage builds, minimal bases (distroless, Alpine, `scratch`), combining package install and cleanup in one `RUN`, `.dockerignore` to exclude `target/`, `.git`, and secrets, and stripping binaries.

---

## 3. Rust-specific images

### 3.1 What does a good Dockerfile for a Rust service look like?

```dockerfile
FROM rust:1-bookworm AS chef
RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json   # cached deps layer
COPY . .
RUN cargo build --release --bin server

FROM gcr.io/distroless/cc-debian12:nonroot
COPY --from=builder /app/target/release/server /server
USER nonroot
ENTRYPOINT ["/server"]
```

`cargo-chef` caches dependency compilation in its own layer, so code changes don't rebuild all crates.

### 3.2 glibc vs musl for Rust containers?

musl (`x86_64-unknown-linux-musl`) produces fully static binaries that run on `scratch` or Alpine. Downsides: musl's allocator is slow under multithreaded load (use `mimalloc`/`jemalloc`), and some C dependencies (OpenSSL) are harder to link (prefer `rustls`). glibc + distroless/cc is the safer default.

### 3.3 What must a `scratch` image include for a Rust HTTP client to work?

CA certificates (for TLS), timezone data if needed, and a statically linked binary. Without `/etc/ssl/certs`, TLS connections fail (or embed roots with `webpki-roots`).

---

## 4. Networking and storage

### 4.1 Which Docker network drivers exist?

`bridge` (default, private network with NAT and port publishing), `host` (shares host network stack, no isolation, best performance), `none`, `overlay` (multi-host, Swarm), and `macvlan`. User-defined bridge networks provide DNS-based service discovery by container name.

### 4.2 Volumes vs bind mounts vs tmpfs?

Volumes are managed by Docker, portable, and the recommended way to persist data. Bind mounts map a host path (good for dev, couples to host layout). tmpfs is in-memory, for secrets or scratch data. The container's writable layer is ephemeral and slow for heavy writes.

### 4.3 What does `EXPOSE` actually do?

Nothing at runtime; it documents the port. Publishing happens with `-p host:container`.

---

## 5. Security

### 5.1 How do you harden a container?

Run as non-root (`USER`), read-only root filesystem, drop all capabilities and add only needed ones, `no-new-privileges`, default seccomp profile, never `--privileged`, resource limits, minimal base images, and don't mount the Docker socket (it's root on the host).

### 5.2 How do you handle secrets?

Never bake them into images or `ENV` in Dockerfiles (visible in layers and `docker inspect`). Use BuildKit secret mounts at build time, and at runtime inject via the orchestrator (Kubernetes Secrets with encryption at rest, AWS Secrets Manager/SSM via IAM roles), preferably as files rather than env vars.

### 5.3 How do you secure the image supply chain?

Pin base images by digest, scan images (Trivy, Grype, ECR scanning) in CI, generate SBOMs (Syft), sign images (cosign/Sigstore) and verify signatures at admission, use a private registry, and rebuild regularly to pick up base image patches.

### 5.4 What are rootless containers and user namespaces?

Running the daemon/runtime as an unprivileged user (rootless Docker, Podman) or mapping container root to an unprivileged host UID via user namespaces, so a container escape doesn't give host root.

---

## 6. Running in production

### 6.1 Why does PID 1 matter in a container?

PID 1 doesn't get default signal handlers (SIGTERM is ignored unless handled) and must reap zombie children. Your app must handle SIGTERM for graceful shutdown, or run under a tiny init (`tini`, `docker run --init`).

### 6.2 What happens on `docker stop`?

SIGTERM is sent to PID 1, then after a grace period (10 s by default) SIGKILL. The app should stop accepting connections, finish in-flight work, flush and exit within the grace period.

### 6.3 How do memory and CPU limits affect applications?

Exceeding the memory limit gets the process OOM-killed (exit code 137). CPU limits cause CFS throttling, which shows up as latency spikes even at low average CPU. Runtimes that size thread pools from CPU count should respect cgroup limits (Rust's `std::thread::available_parallelism` and Tokio do).

### 6.4 What are health checks and how should they be designed?

`HEALTHCHECK` (Docker) or liveness/readiness probes (Kubernetes). Liveness should check only that the process is not stuck (restarting fixes it); readiness checks whether it can serve traffic (dependencies warmed up). Don't make liveness depend on downstream services, or one outage restarts everything.

### 6.5 How do you handle logging from containers?

Log to stdout/stderr in structured JSON; the runtime's log driver or a node agent (Fluent Bit, Vector, CloudWatch agent) ships logs. Avoid writing logs to files inside the container.

### 6.6 How do you debug a distroless container that has no shell?

`kubectl debug` with an ephemeral debug container sharing the process namespace, `docker run --pid=container:<id> --network=container:<id>` with a tools image, or `nsenter` from the host.

---

## 7. Kubernetes basics

### 7.1 Pod vs Deployment vs Service vs Ingress?

A Pod is one or more containers sharing network and volumes. A Deployment manages ReplicaSets for rolling updates. A Service gives a stable virtual IP/DNS and load balances to pods by label. An Ingress (or Gateway API) routes external HTTP traffic to Services.

### 7.2 Requests vs limits in Kubernetes?

Requests are used for scheduling and guarantee resources; limits cap usage. Memory over limit gets OOM-killed; CPU over limit is throttled. Many teams set memory limit = request and omit CPU limits to avoid throttling.

### 7.3 How does Kubernetes scale workloads?

HorizontalPodAutoscaler scales replicas by CPU, memory or custom metrics (e.g. Kafka lag via KEDA). Cluster Autoscaler or Karpenter adds nodes. VerticalPodAutoscaler recommends request sizes.

### 7.4 How do you achieve zero-downtime deployments on Kubernetes?

Rolling updates with `maxUnavailable: 0`, correct readiness probes, graceful SIGTERM handling, a `preStop` sleep so load balancers stop sending traffic before shutdown, PodDisruptionBudgets, and enough replicas spread across zones.
