# Plan Reconciliation

**Date**: 2026-08-30
**Reconciled against**: `docs/MicroTodoSuite evolution plan.md` (the original plan, 2026-08-08)
**Method**: every element below was checked in the working tree of all nine
repositories. Nothing here is marked delivered from a summary, a task register,
or a rendered manifest alone.

## The rule this document applies

The original plan is the baseline. Decisions taken after it modify that
baseline — but only where a decision is actually recorded. **Where no decision
exists, the plan element still stands and is therefore outstanding.** An element
missing without a recorded decision is a gap, not an implicit descope.

Two kinds of gap are distinguished throughout:

- **Missing by definition** — no specification, task, or contract anywhere
  covers it. Nobody has decided what "done" means.
- **Missing by implementation** — a specification defines it; the code,
  manifests, or evidence do not exist yet.

## Recorded decision sources

These four are the only documents that carry authority to modify the plan. Each
was read in full for this reconciliation.

| Source | Date | What it decides |
| --- | --- | --- |
| `docs/MicroTodoSuite evolution plan.md` | 2026-08-08 | The baseline. Sections 1-16 are the full ("expensive") architecture; section 17 is the economical variant. |
| `architecture-review.md` §6, §8 | 2026-08-09 | Nothing is removed from the architecture. Istio, AKS, Route 53, multicloud, and both cost profiles are **deferred targets, not failed designs**. Changes execution order only. |
| Constitution v3.0.0 | 2026-08-24 | Governing authority; economical-to-full safeguards. |
| `specs/009-full-platform-rollout/spec.md` (Assumptions, Scope Boundaries) | 2026-08-24 | Assigns `demo-full` to full staging; fixes the four `/32` operator CIDRs; makes `microtodosuite.online` canonical; declares Redis replication and durable cross-cloud storage **out of scope with the absence stated explicitly**. |

`docs/reconciliation-notes.md` (2026-08-08, gitops) is a historical assessment of
the local pilot, not a decision record. It is superseded by the specifications
that followed it.

**Consequence of `architecture-review.md` §8**: no element of the full profile
has been descoped. Everything absent is outstanding, in one of the two senses
above.

---

## Part A — The original plan, section by section

Legend: **Built** = implementation exists and was located. **Defined** = a
specification or task covers it. **Gap-def** = missing by definition.
**Gap-impl** = missing by implementation.

### §3 Infrastructure (Terraform)

| Plan element | State | Evidence |
| --- | --- | --- |
| Single AWS account, isolation by cluster + VPC | Built | `ops/aws/environments/{dev,demo-full,full-dev,full-prod}/foundation/` |
| Three EKS clusters (dev, staging, prod) | Gap-impl | Roots exist for all three full environments; none applied. Blocked on spec 009 T052-T066. |
| S3 remote backend, key per environment | Built | Five distinct state keys verified |
| **DynamoDB lock table** | **Superseded by decision** | Replaced by Terraform native S3 locking (`use_lockfile = true`). Recorded in `ops/specs/001-aws-dev-foundation/plan.md:32,51` and `ops/aws/environments/dev/backend/README.md:17`. |
| IRSA | Built | `ops/aws/modules/environment-foundation/` |
| Karpenter | Gap-impl | Terraform prerequisites exist (interruption queue, IAM); no `infrastructure/karpenter/` NodePools. Defined in spec 009 T086. |
| Modules vpc, eks, iam, ecr, route53 | Partial | vpc/eks/iam/ecr built inside `environment-foundation`; no separate `route53` module. |
| Azure AKS module, own backend | Gap-impl | No `ops/azure/` directory at all. Defined in spec 009 T125-T126. |
| Infracost in the pipeline | Built | `ops/.github/workflows/aws-full-foundation-checks.yml` (degrades to skip without an API key) |

### §4 Kubernetes platform add-ons

| Add-on | State | Evidence |
| --- | --- | --- |
| **Istio + Kiali** | **Gap-impl** | No mesh anywhere. No Gateway, VirtualService, DestinationRule, PeerAuthentication, or AuthorizationPolicy exists in any repository. The eleven files matching "istio" are `topology-*` component comments and the argo-rollouts vendor bundle. Confirms `architecture-review.md` §6.1, still true a year on. Defined in spec 009 T083. |
| KEDA | Built | `gitops/infrastructure/keda/` |
| cert-manager | Built | `gitops/infrastructure/cert-manager/` |
| External Secrets Operator | Built | `gitops/infrastructure/external-secrets/` |
| Kyverno | Built | `gitops/infrastructure/kyverno/` |
| Chaos Mesh | Gap-impl | No manifests. Defined in spec 009 T087. |
| Falco | Built | `gitops/infrastructure/falco/` |
| OpenCost | Gap-impl | No manifests. Defined in spec 009 T087, T144, T147. |

### §5 GitOps and continuous delivery

| Plan element | State | Evidence |
| --- | --- | --- |
| `clusters/` for eks-dev, staging, prod, aks-dr | Partial | `eks-dev`, `eks-full-{dev,staging,prod}` exist and render zero Applications. **No `clusters/aks-dr/`** — defined in spec 009 T129. |
| `infrastructure/` one folder per add-on | Built | 15 of 26 target components present |
| `apps/<service>/{base,overlays}` | Built | All five services |
| CI opens a PR against gitops on merge | Built | `.github/workflows/promote.yml`; **currently failing** — see Part E |
| dev synchronizes automatically | Built | ArgoCD automated sync |
| Promotion by copying the digest | Built | `gitops/scripts/bump-image.sh` (repaired 2026-08-30, PR #79) |
| Production PR requires manual approval | Built | `gate-prod` job |
| Rollback by `git revert` | Built | Evidence run `20260809T185618Z-git-revert-self-heal` |
| **ArgoCD Notifications to Slack** | **Gap-impl** | The ArgoCD vendor bundle ships the controller; no notification configuration or Slack destination is wired. Defined in specs 006 and 008. |
| **Route 53 latency routing** | **Gap-impl** | No routing resource. Defined in spec 009 T134, T139-T140 (T140 requires separate approval). |

### §6 Deployment strategy

| Plan element | State | Evidence |
| --- | --- | --- |
| Argo Rollouts replacing Deployment | Built | `gitops/infrastructure/argo-rollouts/vendor/v1.9.1/` |
| Dev/staging rolling updates | Built | `components/strategy-*` |
| Prod canary 10/25/50/100 | Gap-impl | `components/strategy-canary/rollout.yaml` exists; the four-step full-profile strategy and its Istio traffic split do not. Defined in spec 009 T100, T110. |
| AnalysisTemplate on error rate and p99 | Gap-impl | Defined in spec 009 T110; no `cluster-analysis-template.yaml`. |
| `aks-dr` receives the promoted version without canary | Gap-impl | Depends on the absent AKS cluster. |

### §7 CI (GitHub Actions)

| Plan element | State | Evidence |
| --- | --- | --- |
| Reusable workflows centralised in `.github` | Built | `ci.yml`, `promote.yml`, `release.yml`, `stack-tests.yml` |
| OIDC to AWS and Azure, no static credentials | Partial / **broken** | OIDC is wired; the AWS role currently refuses `sts:AssumeRoleWithWebIdentity` from service `main` branches. See Part E. |
| Build once, promote between environments | Built | `promote.yml` copies digests |
| SonarQube gate | Built | All five service `ci.yml` |
| Trivy gate | Built | All five service `ci.yml` |
| semantic-release triggering the gitops PR | Built | `release.yml` |
| Syft SBOM | Built | `ci.yml` supply-chain job |
| Cosign signing | Built | `ci.yml` supply-chain job |
| Kyverno verifies signature before admission | Gap-impl | Policies exist; signature-verification policy for the full profile is spec 009 T088. |

### §8 Design patterns

| Plan element | State | Evidence |
| --- | --- | --- |
| Retry, External Configuration documented | Built | Service operational contracts (auth-api, todos-api, log-message-processor) |
| Circuit Breaker | Partial | Implemented in application code (`todos-api/operational.js`). The plan places it **at the Istio level**; that half is Gap-impl. |
| Timeout | Built (application level) | `auth-api/operational.go`, `todos-api/operational.js` |
| **Bulkhead** | **Gap-impl** | Named in spec 009 only; no implementation. |
| **Feature Toggle using OpenFeature** | **Gap-def** | Feature toggles exist as plain environment booleans in three services. **OpenFeature appears in no specification and no task** — only in the constitution. Nobody has decided whether OpenFeature is still the target. |
| **Configuration Server using Spring Cloud Config for users-api** | **Gap-def** | Absent from every repository and every specification. No decision recorded. |

### §9 Testing

| Type | Tool | State | Evidence |
| --- | --- | --- | --- |
| Unit | go test, Jest, JUnit, pytest | Built | All five services |
| Integration | Testcontainers | Built | `todos-api/test/integration/`, `log-message-processor/tests/integration/`, users-api `@SpringBootTest` + H2 |
| Contract | Spectral | Built | `.spectral.yaml` in four services + `contracts/` |
| Contract | Pact | **Partial** | Only `frontend-todos-api.json`. Missing: `frontend -> auth-api` consumer pact; `auth-api <- frontend` and `users-api <- auth-api` provider verification. Spec 007 T011/T012. |
| E2E | Cypress **or** Playwright | Built (Playwright) | `frontend/e2e/`, `e2e.yml`. The plan offered a choice; Playwright was taken. No decision recorded, but the plan permitted either. |
| Performance | Locust | Built | `frontend/e2e/perf/locustfile.py` covering `/login` and todos CRUD, with `baseline.md` |
| Security (DAST) | OWASP ZAP | Built | `frontend/.github/workflows/dast.yml`, nightly at 03:30 |
| Coverage reported to SonarQube | Built | All five service `ci.yml` |

### §10 Observability

| Plan element | State | Evidence |
| --- | --- | --- |
| OpenTelemetry as the single instrumentation layer | Partial | Present in service code; no collector deployed |
| Traces to Jaeger | Built | `gitops/infrastructure/jaeger/vendor/v2.20.0/` |
| Prometheus + Grafana | Built | `gitops/infrastructure/prometheus/vendor/v0.18.0/`, `grafana/vendor/v13.2.0/` |
| **Logs to ELK (Elasticsearch, Logstash, Kibana)** | **Gap-impl** | None vendored. Loki is deployed instead — that is the **economical** substitution from §17, running while the full profile is still the target. Defined in spec 009 T084. |
| **Filebeat as collector** | **Gap-impl** | Grafana Alloy is deployed instead (`infrastructure/loki/alloy.yaml`), again the economical path. Defined in spec 009 T084. |
| Liveness, readiness, startup probes on every Deployment | Partial | Three of five services carry the full contract. frontend and users-api are spec 009 T073/T078 and T076/T081. |
| Alertmanager to Slack | Partial | Alertmanager configured; **Slack destination not wired**. |

### §11 Security

| Plan element | State | Evidence |
| --- | --- | --- |
| Trivy in CI | Built | All five services |
| **Trivy continuously inside the cluster** | **Gap-impl** | Defined in spec 009 T146 (scheduled assessment); not implemented. |
| External Secrets + AWS Secrets Manager | Built | `infrastructure/external-secrets/`, `ops/aws/modules/environment-foundation/` |
| RBAC per namespace and ServiceAccount, IRSA | Built | Spec 005 |
| TLS on every ingress | Gap-impl | cert-manager present; no issuer, ingress, or verified certificate. |
| **Internal mTLS through Istio** | **Gap-impl** | Depends on the absent mesh. |
| Falco runtime detection | Built | `infrastructure/falco/` |
| kube-bench, kube-hunter | Built | `infrastructure/kube-bench/`, `kube-hunter/` |

### §12 Change management

| Plan element | State | Evidence |
| --- | --- | --- |
| semantic-release release notes | Built | `release.yml` |
| Rollback plan per PR | Partial | Practised; not enforced by a template |
| Immutable image tags traceable to the spec | Built | Digest-only promotion |

### §13 Multicloud, DR, chaos, FinOps

| Plan element | State | Evidence |
| --- | --- | --- |
| AKS as secondary cluster | Gap-impl | No `ops/azure/`, no `clusters/aks-dr/`. Spec 009 T118-T130. |
| gitops keeps both clusters on the same version | Gap-impl | Depends on AKS |
| Route 53 routing between clouds | Gap-impl | Spec 009 T134, T139-T140 |
| Active-active latency mode | Gap-impl, gated | Spec 009 T140 requires separate human approval |
| **Redis not replicated — stated in the DR plan** | **Decision recorded** | Spec 009 Scope Boundaries states the absence explicitly, as the plan required. Satisfied by definition; the DR document itself is T138. |
| Chaos Mesh experiments (pod kill, latency, Redis saturation) | Gap-impl | Spec 009 T135; `gitops/experiments/` is empty |
| Full AWS outage game day | Gap-impl | Spec 009 T137 |
| OpenCost + Infracost cost visibility | Partial | Infracost built; OpenCost absent |

### §14 AI agents repository

| Plan element | State | Evidence |
| --- | --- | --- |
| `.claude/{agents,skills,mcp}`, `docs/`, `README.md` | Built | `microservice-app-ai-agents/` matches the planned structure |
| Spec Kit adapted as skills, shared across repositories | Built | `.agents/skills/speckit-*` present in gitops |

### §15 Spec-Driven Development

| Plan element | State | Evidence |
| --- | --- | --- |
| Constitution in docs | Built | `constitution.md` v3.0.0 |
| Specify / Clarify / Plan / Tasks / Implement / Validate | Built | 12 specs across gitops and ops |
| OpenAPI first for REST | Built | auth-api, todos-api, users-api |
| AsyncAPI for Redis events | Built | todos-api, log-message-processor |
| CI verifies the implementation does not deviate | Built | Spectral in service `ci.yml` |

### §17 Economical version

The economical profile is the **currently running system** and the rollback
target for the whole migration. It is not a gap. Its substitutions — Loki for
ELK, Alloy for Filebeat, no mesh, single cluster, SonarCloud — are in force
today and revert as each full-profile element lands.

---

## Part B — Task registers, actual state

Every register below was audited task by task against the working tree. "Stale"
means the register understated delivered work; "accurate" means the unchecked
tasks are genuinely outstanding.

| Repo / spec | Was | Now | Verdict |
| --- | --- | --- | --- |
| gitops 001-local-gitops-pilot | 12/56 | 12/56 | **Stalled, not stale.** `scripts/pilot/` and `bootstrap/local/kind-config.yaml` exist; `assets.lock`, `run-three-clean.sh`, `operator-evaluation.md`, and `newcomer-workflow.sh` do not. Last touched 2026-08-09, when work moved to the cloud. See C5. |
| gitops 002-dual-topology-plumbing | 18/18 | 18/18 | Complete |
| gitops 003-platform-addons | 30/30 | 30/30 | Complete |
| gitops 003-reusable-cicd-delivery | 30/38 | **32/38** | Stale. T003 and T004 were delivered: the org runs both GitHub Apps (`microtodo-gitops-promoter`, `microtodosuite-ci-release`) and SonarCloud with per-service keys (`MicroTodoSuite_auth-api`) through the reusable workflow. |
| gitops 004-service-onboarding | 34/34 | 34/34 | Complete |
| gitops 005-namespace-isolation | 81/95 | 81/95 | **Accurate.** All fourteen unchecked tasks are live-cluster observation ("observe five serial Healthy operations", "activate the fixture by reviewed commit"). Evidence pending, not code. |
| gitops 006-observability | 25/50 | **27/50** | Stale. T001 (`tests/contract/observability.sh`) and T002 (`scripts/managed/verify-observability.sh`) exist. The rest is live evidence plus the version drift in E1. T049 (documentation) is real debt: `docs/platform-addons.md` mentions none of Prometheus, Grafana, Jaeger, or Loki. |
| gitops 007-advanced-testing | 0/27 | **21/27** | Badly stale. Corrected here. |
| gitops 008-security-runtime-hardening | 17/27 | **19/27** | Stale. T001 (`tests/contract/security.sh`) and T002 (`scripts/managed/verify-security.sh`) exist. T027 is real debt: `docs/platform-addons.md` mentions none of Falco, kube-bench, or kube-hunter. |
| gitops 009-full-platform-rollout | 50/162 | 50/162 | Accurate (56 once gitops#78 merges) |
| ops 001-aws-dev-foundation | 30/53 | 30/53 | **Accurate.** Spot-checked: T031 is partial (outputs lack the approved backend policy ARN), T046 is partial (the workflow runs `terraform fmt` only, not ShellCheck/Trivy/Infracost), T025-T027 and T039-T042 reference test files that do not exist. |
| ops 002-full-profile-demo | 0/0 | 0/0 | No task register |

**Totals**: 590 tasks across twelve specs. **327 delivered on `main` today**;
**354 once the reconciliation lands** (gitops#80 ticks 27, verified from its
diff). That leaves **236 outstanding**. Spec 009's 162 was never the whole plan.

> **Recounted 2026-09-07.** The registers now hold **625 tasks**, **391 done**,
> **234 unchecked** — **190 in scope** once spec 001's 44 retired tasks come
> out. The growth is thirteen account-recovery tasks (ops `001` T054–T062,
> gitops `009` T163–T166) written on 2026-09-07 and not yet on `main`; the
> progress is the frontend and users-api operational contracts, plus the
> recovery tasks already executed. The per-member allocation built on this
> count is `full-platform/work-allocation.md` §0.

## Part C — Missing by definition

Nothing defines these, and no decision descopes them. Each needs a decision
before it can be called outstanding *or* dropped.

1. **OpenFeature** (plan §8). Feature toggles ship as plain environment
   booleans in three services. OpenFeature is named in the constitution and in
   no specification or task.
2. **Spring Cloud Config for users-api** (plan §8). Absent from every
   repository and every specification.
3. **Velero** (plan §17, economical column, "optional cold DR"). Named in spec
   005 prose; no task defines it. Lowest priority — the plan itself marked it
   optional.
4. **A separate `route53` Terraform module** (plan §3). Route 53 work is
   defined inside spec 009 against the dev-owner root rather than as the module
   the plan listed. This may be a deliberate simplification, but it is not
   written down anywhere.
5. ~~**Whether the local GitOps pilot is still a target.**~~ **Resolved
   2026-08-30**: retired by maintainer decision (`gitops#82`). The pilot served
   its purpose; its 44 remaining tasks were the formal evidence harness, which
   spec 009's evidence contract superseded. They stay unchecked because they were
   not delivered.
6. **The two undocumented documentation debts.** Spec 006 T049 and spec 008 T027
   both require `docs/platform-addons.md` to cover their components; it covers
   neither observability nor the security trio.

---

## Part D — Missing by implementation

Defined by a specification; code, manifests, or evidence absent. Ordered by what
unblocks the most.

1. **The Phase 4 apply chain** (spec 009 T052-T066). Fifteen ordered steps.
   Everything in US3, US4, and US5 depends on it.
2. **Eleven platform components**: Istio, Kiali, AWS Load Balancer Controller,
   Karpenter, ECK, Elasticsearch, Kibana, Logstash, Filebeat, Chaos Mesh,
   OpenCost.
3. **Two service operational contracts**: frontend (T073/T078), users-api
   (T076/T081).
4. **The Azure estate**: AKS module, DR root, `clusters/aks-dr/`, ACR mirror,
   Key Vault seeding, game day.
5. **Route 53 and ingress TLS**, including the gated active-active step.
6. **The canary strategy and its analysis templates**.
7. **Pact coverage for the two missing pairs** (spec 007 T011/T012).
8. **Slack destinations** for ArgoCD Notifications and Alertmanager.

---

## Part E — Drift and breakage found, requiring a decision

These are not gaps in the plan; they are places where reality and the written
record disagree. None of them should be resolved by quietly editing one side.

### E1 — Spec 006 pins versions that were never shipped

> **Resolved 2026-09-11**: the vendoring is right; spec 006's register is amended
> to the vendored versions. Jaeger v1 reached end of life on 2025-12-31.

| Component | Spec 006 says | Actually vendored |
| --- | --- | --- |
| kube-prometheus | v0.16.0 | v0.18.0 |
| Grafana | 11.7.0 | v13.2.0 |
| Loki | 3.6.0 | v3.7.6 |
| Jaeger | 1.65.0 | **v2.20.0** |

Jaeger is a major-version jump with a different architecture. The specification
text has not been amended and no decision records the change. Either the
specification is stale or the vendoring outran its review — that is a call for
the maintainer, not a text edit.

### E2 — The publisher OIDC role rejects service `main` branches

Merging `auth-api#16` and `log-message-processor#13` on 2026-08-30 produced a
green test and scan run followed by:

```
Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

`release` and all four `promote` jobs were skipped. **No release, no new digest,
and no promotion PR can be produced today.** The subject condition on the shared
publisher role does not cover these repositories' `main`. Spec 009 T061/T062
update that trust policy, which places the fix inside the blocked Phase 4 chain.

> **Still true on 2026-09-07, and now for a second reason.** The five service
> repositories' `.github/workflows/ci.yml` name the retired account
> `916491575487` in both `ecr-repository` and `publisher-role-arn`, so
> `ci / supply-chain` fails at *Configure AWS credentials through GitHub OIDC*
> before the trust policy is ever consulted (todos-api `main` run
> 33467052223). That edit is the release-path unblock and no task names it —
> see E4.

### E3 — Ten promotion PRs were withdrawn

`#64`-`#73` were closed on 2026-08-30. `bump-image.sh` promoted through
`kustomize edit set image`, which re-serialised the whole overlay and duplicated
the comment block inside the embedded `patch:` literal on every run (measured:
10 -> 13 -> 16 -> 19 comment lines over three promotions). Repaired in gitops
PR #79.

### E4 — The AWS account was replaced, and five workflows still point at the old one

Recorded 2026-09-07. Account `916491575487` is retired; the economical dev
backend and foundation were rebuilt in `575172595729` from inspected saved plans
(ops `001` Phase 8, `specs/001-aws-dev-foundation/plan.md` addendum). The prior
state is preserved externally as recovery evidence and is not migrated.

What is not finished, and blocks everything else:

| Item | Where | State |
| --- | --- | --- |
| Active economical ECR and IRSA values | gitops `009` T164 | Uncommitted working tree, 66 files |
| Service workflow account inputs | five service repos, `.github/workflows/ci.yml` | **No task covers this** |
| Republish and promote the five digests | gitops `009` T165 | Not started |
| Merge, bootstrap, verify | ops `001` T060, gitops `009` T166 | Not started |

The middle row is the gap. T165 assumes a working release path; the five
workflow files are what make it work, and rule 5 of §7 of the conventions says
that work needs a task before it is done — either by extending T165's text or by
adding one.

---

## What this reconciliation changed

- `specs/007-advanced-testing/tasks.md`: 21 tasks ticked, six annotated.
- `specs/003-reusable-cicd-delivery/tasks.md`: T003, T004 ticked.
- `specs/006-observability-platform-foundation/tasks.md`: T001, T002 ticked.
- `specs/008-security-runtime-hardening/tasks.md`: T001, T002 ticked.
- This document created.

Deliberately **not** changed:

- **Spec 006's pinned versions** (E1). Amending them is a decision about which
  side is wrong — the specification or the vendoring — and Jaeger v1 to v2 is
  not a bookkeeping edit.
- **Spec 001's status** (C5). Retiring a specification is a decision, not a
  reconciliation.
- Any task whose text could not be fully verified. Where an artifact exists but the
  task asks for more than the artifact provides — ops-001 T031 and T046 — the
  task stays unchecked and the partial delivery is recorded here instead.
