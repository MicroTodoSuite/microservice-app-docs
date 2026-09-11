## Overview
This documentation-only repository centralizes architecture, delivery, and operating guidance for MicroTodoSuite, a cloud-hosted microservices todo application.
It records the constitution, the architecture and delivery decisions, and the full-platform rollout plans; it contains no executable service code.

## Stack
- Documentation: Markdown files with PNG diagrams.
- Runtime, framework, and version: none defined; there is no dependency manifest, lockfile, or documentation-generator configuration.

## Commands
- Build: none defined in the repository.
- Test: none defined; there is no test suite or CI workflow in this repository.
- Local run: none defined because this repository contains only static documentation.

## Structure
- `README.md`: repository purpose and the document index.
- `constitution.md`: the non-negotiable principles and the amendment process.
- `docs/Agile methodology.md`: the Kanban method and the delivery board.
- `docs/Architecture diagrams.md`: the current AWS and Azure architecture, with the retired Azure Container Apps design as history.
- `docs/Branching strategies.md`: the trunk-based branching model, pointing at the binding conventions.
- `docs/Pull request and task tracking conventions.md`: the binding delivery rules.
- `docs/ADR-0001 Multicloud strategy.md` and `docs/Governance and IaC standards program.md`: recorded decisions.
- `full-platform/`: the full-platform rollout plans, capacity limits, and reconciliation.
- `docs/assets/`: diagrams referenced by the Markdown files.

## Conventions
- Write all project artifacts in English and keep diagram links repository-relative under `docs/assets/`.
- Use Trunk-Based Development with short-lived branches and feature flags for incomplete work.
- Treat `specs/<feature>.md`, `plan.md`, and `tasks.md` as the source of truth when they exist.
- Use GitOps exclusively for environment changes: commit them to `microservice-app-gitops` for ArgoCD reconciliation; never run `kubectl apply` against a GitOps-managed cluster.
- Use one AWS account and isolate environments with separate clusters, VPCs, or namespaces.
- Write everything in English — branch names, commit messages, pull-request titles and bodies, review comments, code comments, documentation, and specification text. No bilingual sections. Changing this rule takes a recorded decision in `microservice-app-docs`, not a remark in conversation.
- Open every pull request through `.github/pull_request_template.md` and follow `microservice-app-docs/docs/Pull request and task tracking conventions.md`: one concern per short-lived `<type>/<summary>` branch, a Conventional Commit title with a scope, and every template section filled. Constitution principle 13 makes this binding, not advisory.
- Keep the Spec-Driven Development commit pair intact: `test(<scope>): specify ...` must be committed failing before `feat(<scope>): implement ...`. Never squash the pair; the failing-test commit is the evidence the cycle was followed.
- Track every task. Name in the pull-request body the task IDs it advances, qualified by repository and spec, and update `tasks.md` in that same pull request rather than a follow-up. Mark a task `[X]` only after locating and inspecting its named artifact — never from a summary, a green check, a rendered manifest, or recollection. Annotate partial delivery instead of ticking it; work no register covers either gains a task or records in the PR body why none applies.
- Reconcile, never quietly edit, when a register and reality disagree: a specification that pins a version nobody shipped is a maintainer decision, and `microservice-app-docs/full-platform/plan-reconciliation.md` is the worked example.
- Never merge with `--admin`, force-push to `main`, disable a branch protection rule to land one's own work, or approve one's own pull request. An AI agent may open, describe, and update a pull request; it may never approve one and never author an acceptance or approval artifact — only a named human unlocks a gate.
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
