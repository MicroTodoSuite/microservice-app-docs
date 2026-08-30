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
- Write pull-request bodies bilingually: every section in English, then repeated under a `## Español` heading with the same content, not a summary. Titles, commits, code comments, documentation, and specification text stay English-only. As an AI agent you write both halves yourself.
- Open every pull request through `.github/pull_request_template.md` and follow `microservice-app-docs/docs/Pull request and task tracking conventions.md`: one concern per short-lived `<type>/<summary>` branch, a Conventional Commit title with a scope, and every template section filled. Constitution principle 13 makes this binding, not advisory.
- Keep the Spec-Driven Development commit pair intact: `test(<scope>): specify ...` must be committed failing before `feat(<scope>): implement ...`. Never squash the pair; the failing-test commit is the evidence the cycle was followed.
- Track every task. Name in the pull-request body the task IDs it advances, qualified by repository and spec, and update `tasks.md` in that same pull request rather than a follow-up. Mark a task `[X]` only after locating and inspecting its named artifact — never from a summary, a green check, a rendered manifest, or recollection. Annotate partial delivery instead of ticking it; work no register covers either gains a task or records in the PR body why none applies.
- Reconcile, never quietly edit, when a register and reality disagree: a specification that pins a version nobody shipped is a maintainer decision, and `microservice-app-docs/full-platform/plan-reconciliation.md` is the worked example.
- Never merge with `--admin`, force-push to `main`, disable a branch protection rule to land your own work, or approve your own pull request. As an AI agent you may open, describe, and update a pull request; you may never approve one and never author an acceptance or approval artifact — only a named human unlocks a gate.
- Report outcomes faithfully in commits and pull-request bodies: name what is red, say what was skipped, and correct an earlier claim that turns out to be wrong rather than leaving the record wrong.

## Notes for the Kubernetes migration
- This repository exposes no ports and defines no environment variables. Verify both in each application repository before writing Kubernetes workloads or Services.
- No Dockerfile, Azure Container Apps configuration, Terraform source, Kubernetes manifest, or GitOps manifest is checked in here; review those artifacts in their owning repositories.
- The implemented diagram contains `frontend`, `auth-api`, `users-api`, `todos-api`, `log-message-processor`, Redis, Zipkin, Prometheus, Grafana, and an nginx Prometheus exporter.
- The documentation describes synchronous service calls, Redis-backed asynchronous log processing, distributed tracing through Zipkin, and metrics collection through Prometheus with Grafana dashboards.
- Review Azure Container Apps scaling (`min_replicas` and `max_replicas`), per-service container isolation, Azure Container Registry image pulls, and Azure Storage-backed Terraform state when mapping the platform to Kubernetes and AWS.
- Do not migrate initial-proposal-only Azure API Management, Cosmos DB, Key Vault, Function App, VNet/subnet, or Network Security Group components without confirming that they are still required; they are absent from the implemented diagram.
- Verify where retry and circuit-breaker behavior is implemented; the documentation claims these patterns but provides no application or platform configuration.
- Deliver all Kubernetes environment changes through `microservice-app-gitops` and ArgoCD, preserving environment isolation inside the single AWS account.
