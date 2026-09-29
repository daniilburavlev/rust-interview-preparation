# Deployment and CI/CD: Questions Only

Questions without answers, for self-testing. Model answers are in [deployment-cicd.md](deployment-cicd.md).

## 1. CI/CD fundamentals

- 1.1 Continuous integration vs continuous delivery vs continuous deployment?
- 1.2 Trunk-based development vs GitFlow?
- 1.3 What are the DORA metrics?
- 1.4 What does "build once, deploy many" mean?

## 2. Pipeline design

- 2.1 What stages would you put in a pipeline for a backend service?
- 2.2 How do you make pipelines fast?
- 2.3 How do you deal with flaky tests?
- 2.4 How do you run integration tests against databases and Kafka in CI?
- 2.5 What are merge queues and why use them?

## 3. Rust in CI

- 3.1 What checks should a Rust CI pipeline run?
- 3.2 How do you speed up Rust builds in CI?
- 3.3 How do you release Rust binaries and crates?
- 3.4 Why pin the Rust toolchain?

## 4. Deployment strategies

- 4.1 Rolling vs blue/green vs canary?
- 4.2 What is progressive delivery and automated canary analysis?
- 4.3 Feature flags vs deployments?
- 4.4 How do you ensure old and new versions can run side by side?
- 4.5 How do you roll back safely?

## 5. Database and schema changes

- 5.1 What is the expand/contract (parallel change) pattern?
- 5.2 How do you run migrations in a deployment pipeline?
- 5.3 Which migrations are dangerous on large tables?

## 6. GitOps and configuration

- 6.1 What is GitOps?
- 6.2 Push-based vs pull-based deployment?
- 6.3 Helm vs Kustomize?
- 6.4 How do you manage configuration and secrets per environment?

## 7. Security and supply chain

- 7.1 How do you secure a CI/CD pipeline?
- 7.2 What is SLSA and software supply chain security?
- 7.3 What should be scanned in the pipeline?

## 8. Operating releases

- 8.1 What should you check right after a deployment?
- 8.2 How do you handle a failed production deploy?
- 8.3 How do you deploy a Kafka consumer or stateful service safely?
- 8.4 How do you manage multiple environments without drift?
