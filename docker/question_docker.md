# Docker and Containers: Questions Only

Questions without answers, for self-testing. Model answers are in [docker.md](docker.md).

## 1. Fundamentals

- 1.1 Container vs virtual machine?
- 1.2 Which Linux features make containers work?
- 1.3 What are Docker Engine, containerd and runc? What is OCI?
- 1.4 Image vs container vs layer?

## 2. Images and Dockerfiles

- 2.1 `CMD` vs `ENTRYPOINT`? Shell vs exec form?
- 2.2 How does layer caching work and how do you order instructions?
- 2.3 What are multi-stage builds?
- 2.4 `COPY` vs `ADD`?
- 2.5 What is BuildKit and which features matter?
- 2.6 How do you build multi-architecture images?
- 2.7 How do you keep images small?

## 3. Rust-specific images

- 3.1 What does a good Dockerfile for a Rust service look like?
- 3.2 glibc vs musl for Rust containers?
- 3.3 What must a `scratch` image include for a Rust HTTP client to work?

## 4. Networking and storage

- 4.1 Which Docker network drivers exist?
- 4.2 Volumes vs bind mounts vs tmpfs?
- 4.3 What does `EXPOSE` actually do?

## 5. Security

- 5.1 How do you harden a container?
- 5.2 How do you handle secrets?
- 5.3 How do you secure the image supply chain?
- 5.4 What are rootless containers and user namespaces?

## 6. Running in production

- 6.1 Why does PID 1 matter in a container?
- 6.2 What happens on `docker stop`?
- 6.3 How do memory and CPU limits affect applications?
- 6.4 What are health checks and how should they be designed?
- 6.5 How do you handle logging from containers?
- 6.6 How do you debug a distroless container that has no shell?

## 7. Kubernetes basics

- 7.1 Pod vs Deployment vs Service vs Ingress?
- 7.2 Requests vs limits in Kubernetes?
- 7.3 How does Kubernetes scale workloads?
- 7.4 How do you achieve zero-downtime deployments on Kubernetes?
