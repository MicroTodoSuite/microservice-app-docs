## Overview
This documentation-only repository centralizes architecture, delivery, and operating guidance for MicroTodoSuite, a cloud-hosted microservices todo application.
It records agile and branching decisions plus the initial and implemented Azure architectures; it contains no executable service code.

## Stack
- Documentation: Markdown files with PNG diagrams.
- Runtime, framework, and version: none defined; there is no dependency manifest, lockfile, or documentation-generator configuration.

## Commands
- Build: none defined in the repository.
- Test: none defined; there is no test suite or CI workflow in this repository.
- Local run: none defined because this repository contains only static documentation.

## Structure
- `README.md`: repository purpose, document index, and navigation links.
- `docs/Agile methodology.md`: the documented dual-Kanban delivery model.
- `docs/Architecture diagrams.md`: rationale for the initial and implemented Azure architectures.
- `docs/Branching strategies.md`: the project's historical development and operations branching guidance.
- `docs/Report.md`: placeholder for the technical report; currently empty.
- `docs/assets/`: architecture and branching diagrams referenced by the Markdown files.

## Conventions
- Write all project artifacts in English and keep diagram links repository-relative under `docs/assets/`.
- Use Trunk-Based Development with short-lived branches and feature flags for incomplete work.
- Treat `specs/<feature>.md`, `plan.md`, and `tasks.md` as the source of truth when they exist.
- Use GitOps exclusively for environment changes: commit them to `microservice-app-gitops` for ArgoCD reconciliation; never run `kubectl apply` against a GitOps-managed cluster.
- Use one AWS account and isolate environments with separate clusters, VPCs, or namespaces.
- The suite-wide Trunk-Based Development rule supersedes the GitHub Flow guidance in `docs/Branching strategies.md`.

## Notes for the Kubernetes migration
- This repository exposes no ports and defines no environment variables. Verify both in each application repository before writing Kubernetes workloads or Services.
- No Dockerfile, Azure Container Apps configuration, Terraform source, Kubernetes manifest, or GitOps manifest is checked in here; review those artifacts in their owning repositories.
- The implemented diagram contains `frontend`, `auth-api`, `users-api`, `todos-api`, `log-message-processor`, Redis, Zipkin, Prometheus, Grafana, and an nginx Prometheus exporter.
- The documentation describes synchronous service calls, Redis-backed asynchronous log processing, distributed tracing through Zipkin, and metrics collection through Prometheus with Grafana dashboards.
- Review Azure Container Apps scaling (`min_replicas` and `max_replicas`), per-service container isolation, Azure Container Registry image pulls, and Azure Storage-backed Terraform state when mapping the platform to Kubernetes and AWS.
- Do not migrate initial-proposal-only Azure API Management, Cosmos DB, Key Vault, Function App, VNet/subnet, or Network Security Group components without confirming that they are still required; they are absent from the implemented diagram.
- Verify where retry and circuit-breaker behavior is implemented; the documentation claims these patterns but provides no application or platform configuration.
- Deliver all Kubernetes environment changes through `microservice-app-gitops` and ArgoCD, preserving environment isolation inside the single AWS account.
