# MicroTodoSuite Documentation

This repository holds the governing documents of MicroTodoSuite: the
constitution, the architecture and delivery decisions, and the plans for the
full-platform rollout. GaCode Solutions designs, builds, and operates the
platform for its client, Lexfield Legal. The repository contains no executable
code.

## Governing documents

| Document | Purpose |
| --- | --- |
| [Constitution](constitution.md) | The non-negotiable principles, the operational baseline, and the amendment process. |
| [Pull request and task tracking conventions](docs/Pull%20request%20and%20task%20tracking%20conventions.md) | The binding branch, commit, pull-request, language, and task-tracking rules. Constitution principle 13 makes them mandatory. |
| [ADR-0001: Multicloud strategy](docs/ADR-0001%20Multicloud%20strategy.md) | Why AWS is the single primary platform and Azure an independent recovery domain. |
| [Governance and IaC standards program](docs/Governance%20and%20IaC%20standards%20program.md) | The 2026 governance and infrastructure-as-code program and its recorded decisions. |

## Architecture and delivery

| Document | Purpose |
| --- | --- |
| [Architecture](docs/Architecture%20diagrams.md) | The current platform architecture and the retired Azure Container Apps design. |
| [Branching strategy](docs/Branching%20strategies.md) | The branching model used by every repository. |
| [Agile methodology](docs/Agile%20methodology.md) | The Kanban method and the delivery board. |
| [Evolution plan](docs/MicroTodoSuite%20evolution%20plan.md) | The consolidated plan for the migration from Azure Container Apps to AWS EKS, with the economical and full profiles. |
| [Architecture review](architecture-review.md) | The 2026-08-09 review of the target architecture against the validated workspace state. |

## Full-platform rollout

| Document | Purpose |
| --- | --- |
| [Rollout index](full-platform/rollout-index.md) | The stages of the full-platform rollout, their owners, and their order. |
| [Infrastructure execution plan](full-platform/infrastructure-execution-plan.md) | The sequencing of the infrastructure lane. |
| [Capacity, regions, and the account parameter](full-platform/capacity-and-regions.md) | The limits that let both profiles run at once in one account, and the region placement. |
| [Economical safety boundary](full-platform/economical-safety-boundary.md) | What protects the economical platform while the full profile is built. |
| [Plan reconciliation](full-platform/plan-reconciliation.md) | The task registers reconciled against delivered reality. |
| [Work allocation](full-platform/work-allocation.md) | The remaining work distributed across the team lanes. |

## Contributing

Every change follows the pull request and task tracking conventions and the
repository's pull-request template. Documents are written in English, in a
professional register and in the third person, without emoji. A change to the
constitution follows its amendment process.
