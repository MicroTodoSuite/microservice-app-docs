# Full-Platform Rollout Index

This index coordinates the staged implementation governed by
`microservice-app-gitops/specs/009-full-platform-rollout/`. The specification,
plan, contracts, and task list are authoritative. A stage cannot advance when
its evidence bundle is missing, stale, failing, or blocked.

## Repository Ownership

| Repository | Owned scope | Required reviewer role |
| --- | --- | --- |
| `microservice-app-gitops` | Specification, Kubernetes desired state, cluster roots, bootstrap guard, evidence schema and collectors | GitOps maintainer |
| `microservice-app-ops` | AWS/Azure Terraform foundations, state boundaries, cloud identity, DNS and plan-only workflows | Infrastructure maintainer |
| `.github` | Reusable build, release, promotion, image-mirror, secret-seed and recurring-security workflows | Organization automation maintainer |
| `microservice-app-auth-api` | Auth health, resilience, telemetry, contracts and CI caller | Auth service maintainer |
| `microservice-app-frontend` | Frontend health, runtime configuration, telemetry, E2E/performance/DAST and CI caller | Frontend maintainer |
| `microservice-app-log-message-processor` | Consumer health, Redis resilience, telemetry, AsyncAPI and CI caller | Log-processor maintainer |
| `microservice-app-todos-api` | API health, Redis resilience, telemetry, OpenAPI/AsyncAPI and CI caller | Todos service maintainer |
| `microservice-app-users-api` | Actuator health, authentication, telemetry, OpenAPI and CI caller | Users service maintainer |
| `microservice-app-docs` | Stage register, safety boundaries, operational procedures, status and handoff | Documentation maintainer plus the owner of the documented system |

## Stage and Pull-Request Order

| Order | Stage | Primary owner | Required predecessors | Merge or execution gate |
| ---: | --- | --- | --- | --- |
| 1 | Specification, evidence mechanics and Azure preflight | GitOps | Constitution 3.0.0 | GitOps and ops tests pass; exact reviewed head is merged |
| 2 | Foundational Terraform and profile compatibility | Ops and GitOps | Stage 1 | Refreshed dev plan is literally `0 to add, 0 to change, 0 to destroy`; economical golden renders are unchanged |
| 3 | Economical baseline and blocked-stage drill | GitOps | Stage 2 | Live economical health and drift evidence pass before and after the drill |
| 4 | Centralized egress and three full AWS foundations | Ops | Stage 3 | Separate plans, quotas, Infracost, rollback and exact-plan approvals; apply order is egress, full dev, full prod, staging prerequisites, then dev-owned shared trust |
| 5 | Activation-empty EKS GitOps roots and bootstrap | GitOps | Stage 4 | Roots are merged first; each new cluster receives only the two audited bootstrap mutations |
| 6 | Platform images, controllers and service operational contracts | Organization workflows, services and GitOps | Stage 5 | Complete signed image graph, service tests and dependency-wave acceptance pass before business activation |
| 7 | Immutable promotion and AWS production canary | Organization workflows, services and GitOps | Stage 6 | One digest passes all gates and a deliberate canary failure restores stable service |
| 8 | AKS DR foundation, mirror and secret seed | Ops, organization workflows and GitOps | Stage 7 | Authenticated Azure discovery, exact approved plan, independent ArgoCD, value-blind secret parity and ECR/ACR graph equality pass |
| 9 | Game day and optional production routing | GitOps and Ops | Stage 8 | DR game day passes; production routing still requires a separate named approval for the exact Route 53 plan and cost |
| 10 | Final evidence and handoff | All owners | Selected prior stages | All mandatory tests, scans, live checks, rollback evidence and requirement mappings pass |

Within a stage, tests merge before or with their implementation. Cross-repository
PRs reference the same stage ID and are merged in dependency order. A later PR
must not assume an earlier branch: it references only a predecessor already
merged to `main` and records that merge SHA.

## Approval Roles

- Every PR requires a maintainer of the repository being changed.
- Terraform plans require the infrastructure maintainer and the operator who
  accepts the exact cost and availability trade-off. Approval applies only to
  the saved-plan checksum reviewed at that time.
- GitOps activation requires a GitOps maintainer and the owner of the affected
  environment or service. Approval must match the final head SHA.
- Service contract or release-gate changes require the owning service
  maintainer and the organization automation maintainer when shared workflows
  change.
- Azure DR requires the approved subscription owner in addition to the
  infrastructure and GitOps maintainers.
- Enabling `app.microtodosuite.online` traffic requires a separate named traffic
  owner after the game day. Prior architecture, plan, apply, or PR approval does
  not satisfy this gate.
- Constitution changes follow the constitution's broader multi-maintainer
  amendment process and cannot be approved by an automation agent.

## Escalation and Stop Path

1. Stop the current stage on a failed plan, inaccessible state, unexpected
   destroy/replacement, singleton duplication, quota or cost excess, DNS record
   loss risk, unreviewed mutation, missing evidence, or economical regression.
2. Mark the stage `blocked` in its evidence bundle and record the exact failing
   command, artifact checksum and owning role without including credentials or
   secret values.
3. Notify the primary owner and the owner of the failed dependency. Security,
   state, DNS and production-traffic failures also notify the infrastructure or
   traffic owner respectively.
4. Use only the reviewed rollback defined by the stage. Do not force-push,
   bypass branch protection, use `--admin`, apply a new Terraform plan under an
   old approval, or mutate a GitOps-managed cluster directly.
5. Re-open the stage with a new reviewed head or saved plan, recapture the
   economical pre-baseline, and repeat every invalidated gate.

The working economical platform remains the rollback target throughout this
rollout and is never a cleanup target for a failed full-profile stage.
