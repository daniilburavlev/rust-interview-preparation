# Deployment and CI/CD: Senior Interview Questions

Pipelines, release strategies, and operating deployments safely, with notes for Rust projects.

## Contents

1. [CI/CD fundamentals](#1-cicd-fundamentals)
2. [Pipeline design](#2-pipeline-design)
3. [Rust in CI](#3-rust-in-ci)
4. [Deployment strategies](#4-deployment-strategies)
5. [Database and schema changes](#5-database-and-schema-changes)
6. [GitOps and configuration](#6-gitops-and-configuration)
7. [Security and supply chain](#7-security-and-supply-chain)
8. [Operating releases](#8-operating-releases)

---

## 1. CI/CD fundamentals

### 1.1 Continuous integration vs continuous delivery vs continuous deployment?

CI: merge small changes frequently, each validated by automated builds and tests. Continuous delivery: every change that passes the pipeline is releasable, with a manual approval to deploy. Continuous deployment: every passing change goes to production automatically.

### 1.2 Trunk-based development vs GitFlow?

Trunk-based: short-lived branches merged to `main` daily, incomplete features hidden behind flags; fits continuous deployment and reduces merge pain. GitFlow: long-lived `develop`/release branches; suits versioned releases (libraries, mobile) but slows feedback.

### 1.3 What are the DORA metrics?

Deployment frequency, lead time for changes, change failure rate, and time to restore service (plus reliability in newer reports). They measure delivery performance; elite teams deploy on demand with low failure rates and fast recovery.

### 1.4 What does "build once, deploy many" mean?

Build one immutable artifact (container image tagged with the commit SHA or digest) and promote the same artifact through staging and production, varying only configuration. Rebuilding per environment risks deploying something different from what was tested.

---

## 2. Pipeline design

### 2.1 What stages would you put in a pipeline for a backend service?

Lint and format check → build → unit tests → integration tests (with real dependencies in containers) → security scans (dependencies, image, secrets) → build and push image → deploy to staging → smoke/e2e tests → deploy to production progressively → post-deploy verification.

### 2.2 How do you make pipelines fast?

Cache dependencies and build outputs, run independent jobs in parallel, fail fast with cheap checks first, only run affected tests in monorepos (path filters, build graphs), use bigger or self-hosted runners for heavy builds, and keep integration tests focused.

### 2.3 How do you deal with flaky tests?

Treat them as bugs: track flakiness, quarantine only with an owner and deadline, fix root causes (timing assumptions, shared state, test order dependence, real network). Blind automatic retries hide real race conditions.

### 2.4 How do you run integration tests against databases and Kafka in CI?

Spin up dependencies with service containers or Testcontainers, use isolated schemas or databases per test, run migrations as part of setup, and seed data explicitly. Avoid depending on shared long-lived test environments.

### 2.5 What are merge queues and why use them?

A merge queue tests each PR on top of the latest `main` plus the PRs queued ahead of it before merging, preventing "green PRs that break main together" in busy repositories.

---

## 3. Rust in CI

### 3.1 What checks should a Rust CI pipeline run?

`cargo fmt --check`, `cargo clippy --all-targets --all-features -- -D warnings`, `cargo test` (or `cargo nextest`), doc tests, `cargo deny check` / `cargo audit`, MSRV check for libraries, `cargo semver-checks` for published crates, and `cargo miri test` for crates with unsafe code.

### 3.2 How do you speed up Rust builds in CI?

`Swatinem/rust-cache` or `sccache` with a remote backend, `cargo-chef` in Docker builds, a faster linker (`mold`/`lld`), reducing debug info in CI (`CARGO_PROFILE_DEV_DEBUG=0`), disabling incremental compilation in CI (`CARGO_INCREMENTAL=0`), `cargo nextest` for parallel tests, and splitting the workspace so changes rebuild less.

### 3.3 How do you release Rust binaries and crates?

Tag-driven workflows that cross-compile for targets (`cross`, `cargo-zigbuild`, or native runners per OS/arch), generate checksums and SBOMs, publish GitHub releases, and publish crates with `cargo publish` via tools like `release-plz` or `cargo-release` that handle versions and changelogs.

### 3.4 Why pin the Rust toolchain?

A `rust-toolchain.toml` file makes local and CI builds use the same compiler and components; new clippy lints or compiler changes won't break CI unexpectedly. Update it deliberately.

---

## 4. Deployment strategies

### 4.1 Rolling vs blue/green vs canary?

- **Rolling**: replace instances gradually; cheap, but old and new versions run together and rollback is another rolling deploy.
- **Blue/green**: deploy a full new environment and switch traffic at once; instant rollback, doubles capacity during deploy.
- **Canary**: send a small percentage of traffic to the new version, compare metrics, then increase; limits blast radius, requires good metrics and traffic splitting.

### 4.2 What is progressive delivery and automated canary analysis?

Automating canary promotion based on SLO metrics (error rate, latency) compared to the baseline, with automatic rollback on regression. Tools: Argo Rollouts, Flagger, AWS CodeDeploy, Spinnaker/Kayenta.

### 4.3 Feature flags vs deployments?

Deploy puts code into production; a feature flag controls release to users. Flags enable dark launches, gradual rollouts, kill switches and A/B tests. Cost: flag debt, so remove flags after rollout and test both paths.

### 4.4 How do you ensure old and new versions can run side by side?

Backward and forward compatible APIs and events (additive changes, tolerant readers), expand/contract database migrations, and not changing the meaning of existing fields. Every deployment strategy except a full stop-the-world deploy requires this.

### 4.5 How do you roll back safely?

Keep previous artifacts and deploy them through the same pipeline, keep DB migrations backward compatible so rollback doesn't need a down-migration, prefer roll-forward for data changes, and practice rollbacks. Feature flags are often the fastest rollback.

---

## 5. Database and schema changes

### 5.1 What is the expand/contract (parallel change) pattern?

To rename a column: (1) expand: add the new column, write to both; (2) backfill; (3) switch reads to the new column; (4) contract: stop writing and drop the old column in a later release. Each step is deployable and reversible independently.

### 5.2 How do you run migrations in a deployment pipeline?

As a separate, versioned step before the app rollout (a Kubernetes Job or pipeline step), using tools like `sqlx migrate`, `refinery`, Flyway or Liquibase. Migrations must be compatible with the currently running version. Avoid running migrations concurrently from every app instance at startup.

### 5.3 Which migrations are dangerous on large tables?

Adding a column with a volatile default (table rewrite in older engines), adding indexes without `CONCURRENTLY` (Postgres), changing column types, adding NOT NULL constraints without a validated check first, and long-running transactions holding locks. Use `lock_timeout`, online schema change tools (`gh-ost`, `pg-osc`), and batched backfills.

---

## 6. GitOps and configuration

### 6.1 What is GitOps?

Git is the source of truth for the desired state of environments. An agent in the cluster (Argo CD, Flux) pulls and reconciles the actual state to match Git, detecting and correcting drift. Deploys become pull requests; rollbacks are reverts; audit history comes for free.

### 6.2 Push-based vs pull-based deployment?

Push: CI has credentials to the cluster and applies changes. Pull (GitOps): an in-cluster agent pulls changes, so CI doesn't need cluster credentials and drift is continuously corrected.

### 6.3 Helm vs Kustomize?

Helm: templated charts with values, packaging and release history; powerful but templates get complex. Kustomize: template-free overlays patching base YAML per environment; built into `kubectl`. Many teams use Helm for third-party apps and Kustomize or Helm for their own.

### 6.4 How do you manage configuration and secrets per environment?

Twelve-factor style: config via environment or mounted files, not baked into images. Secrets come from a secret manager (AWS Secrets Manager, Vault) via External Secrets Operator or Sealed Secrets/SOPS for encrypted-in-Git. Validate config at startup and fail fast.

---

## 7. Security and supply chain

### 7.1 How do you secure a CI/CD pipeline?

Least-privilege tokens, OIDC to cloud providers instead of static keys, pin third-party actions by commit SHA, restrict who can modify workflows, protect `main` with required reviews and checks, isolate untrusted PR builds (no secrets for forks), and audit runner images.

### 7.2 What is SLSA and software supply chain security?

SLSA (Supply-chain Levels for Software Artifacts) is a framework for build integrity: builds on hosted, isolated infrastructure producing signed provenance. Combine with SBOMs, image signing (cosign), dependency scanning, and admission policies that only allow signed images from trusted builders.

### 7.3 What should be scanned in the pipeline?

Dependencies (`cargo audit`/`cargo deny`, Dependabot), container images (Trivy, Grype), IaC (Checkov, tfsec), secrets in code (gitleaks, GitHub secret scanning), and SAST (CodeQL, Semgrep).

---

## 8. Operating releases

### 8.1 What should you check right after a deployment?

Error rates, latency percentiles, saturation, business metrics (orders, sign-ups), logs for new error types, and dependency health, compared to the pre-deploy baseline. Automate as much as possible and define rollback criteria before deploying.

### 8.2 How do you handle a failed production deploy?

Stop the rollout, roll back or disable the feature flag first to restore service, communicate status, then investigate. Afterward, run a blameless postmortem with timeline, root cause, and action items (e.g. a missing test or canary metric).

### 8.3 How do you deploy a Kafka consumer or stateful service safely?

Consumers: ensure new versions handle old message formats, use cooperative rebalancing and static membership to reduce rebalance storms, and commit offsets on graceful shutdown. Stateful services: StatefulSets with ordered updates, PodDisruptionBudgets, and data migrations decoupled from deploys.

### 8.4 How do you manage multiple environments without drift?

Same artifact and same IaC modules across environments with only parameters differing, GitOps reconciliation, ephemeral preview environments per PR where affordable, and regularly syncing production-like data (anonymized) into staging.
