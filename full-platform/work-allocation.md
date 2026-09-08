# Work Allocation

**Date**: 2026-09-07
**Basis**: every `tasks.md` register in `microservice-app-gitops` and
`microservice-app-ops`, counted on 2026-09-07 including the account-recovery
work not yet on `main`, classified from `full-platform/plan-reconciliation.md`.

> **Sections 1 through 2d are the 2026-08-30 revision.** They are kept for the
> reasoning they record, not for their numbers. Three things changed since:
> the AWS account was replaced, thirteen recovery tasks were added, and the
> service operational contracts landed. **Section 0 is the current balance and
> the current allocation.**

The team split the project three ways at the start — infrastructure, CI/CD,
observability and monitoring. This document answers two questions with data
rather than impression: how even that split turned out, and how the remaining
work divides so each member has a boundary they can act on alone.

---

## 0. Balance and allocation, 2026-09-07

### 0.1 Every register, counted

| Repo / spec | Done | Total | Outstanding |
| --- | --- | --- | --- |
| gitops `001-local-gitops-pilot` | 12 | 56 | *(44 — retired by `gitops#82`, out of scope)* |
| gitops `002-dual-topology-plumbing` | 18 | 18 | 0 |
| gitops `003-platform-addons` | 30 | 30 | 0 |
| gitops `003-reusable-cicd-delivery` | 32 | 38 | 6 |
| gitops `004-service-onboarding` | 34 | 34 | 0 |
| gitops `005-namespace-isolation` | 81 | 95 | 14 |
| gitops `006-observability-platform-foundation` | 27 | 50 | 23 |
| gitops `007-advanced-testing` | 21 | 27 | 6 |
| gitops `008-security-runtime-hardening` | 19 | 27 | 8 |
| gitops `009-full-platform-rollout` | 57 | 166 | 109 |
| ops `001-aws-dev-foundation` | 38 | 62 | 24 |
| ops `002-full-profile-demo` | 22 | 22 | 0 |
| **Total** | **391** | **625** | **234 unchecked — 190 in scope** |

Two qualifications on those numbers:

- **Thirteen of the tasks are not on `main` yet.** ops `001` T054–T062 and
  gitops `009` T163–T166 were written on 2026-09-07 for the account recovery
  and live on `chore/migrate-aws-account-575172595729` in both repositories.
- **Four more close on review.** gitops PR #85 ticks 009 T073, T076, T078 and
  T081 against the frontend and users-api contracts merged on 2026-08-31. Its
  three checks are green and it is waiting for a reviewer, not for work. With
  it merged the in-scope figure is **186**.

### 0.2 What changed since the 2026-08-30 revision

1. **The AWS account was replaced.** `916491575487` is retired. The economical
   dev backend and foundation now live in `575172595729`, applied from
   inspected saved plans (ops `001` T054–T059, T061–T062). The prior state is
   kept as an external recovery backup and is not migrated.
2. **The GitOps side of that recovery is unfinished.** gitops `009` T164 is in
   the working tree — 66 files repointing the active economical ECR and IRSA
   values — and T165 (republish and promote the five digests) and T166 (merge,
   bootstrap, verify) are untouched.
3. **The service operational contracts landed** for frontend and users-api,
   which were the last two of five.
4. **CVE-2026-14456 is remediated** in frontend and todos-api. It no longer
   blocks anything.
5. **The release path is still completely blocked, now for a new reason.** All
   five service repositories still name the retired account in
   `.github/workflows/ci.yml`, for both `ecr-repository` and
   `publisher-role-arn`. `ci / supply-chain` fails at *Configure AWS
   credentials through GitHub OIDC*, and `release` plus all four `promote` jobs
   are skipped — observed on the todos-api `main` run 33467052223 and the
   users-api `main` run of 2026-08-31. No release, no digest, no promotion PR
   can be produced until those five files change.

### 0.3 Recorded decision — the infrastructure lane stays whole

**Maintainer decision, recorded 2026-09-07.** Esteban Gaviria keeps the whole
infrastructure lane, allocated by subject matter, rather than the 44-operate /
34-author division §2d recommends.

§2d is retained as analysis and its measurement stands — infrastructure carries
the most weight per task, and a third of the lane is authoring that needs no
cluster. It is no longer a proposal.

### 0.4 The allocation

| Member | Area | Tasks | Share |
| --- | --- | --- | --- |
| Esteban Gaviria | Infrastructure | **82** | 43 % |
| Santiago Valencia | Observability and monitoring | **60** | 32 % |
| Juan Manuel Díaz | CI/CD and delivery | **48** | 25 % |
| | | **190** | |

Santiago's figure becomes 56 when gitops PR #85 merges.

#### Esteban Gaviria — Infrastructure, 82 tasks

| Register | N | Tasks |
| --- | --- | --- |
| gitops `009-full-platform-rollout` | 52 | T052–T071, T083, T086, T089, T091, T093–T098, T118–T134, T154, T156, T164–T166 |
| ops `001-aws-dev-foundation` | 24 | T025–T028, T030–T031, T035–T036, T038–T050, T052–T053, T060 |
| gitops `005-namespace-isolation` | 6 | T042–T043, T068, T075, T092, T095 |

#### Santiago Valencia — Observability and monitoring, 60 tasks

| Register | N | Tasks |
| --- | --- | --- |
| gitops `006-observability-platform-foundation` | 23 | T003–T007, T009–T010, T014a, T021–T022, T024–T026, T030–T031, T038–T039, T044–T049 |
| gitops `009-full-platform-rollout` | 21 | T073, T076, T078, T081, T084–T085, T087–T088, T090, T135–T141, T143–T144, T146–T147, T158 |
| gitops `005-namespace-isolation` | 8 | T064–T065, T069, T072, T079, T082, T089–T090 |
| gitops `008-security-runtime-hardening` | 8 | T003–T004, T013, T018, T023, T025–T027 |

#### Juan Manuel Díaz — CI/CD and delivery, 48 tasks

| Register | N | Tasks |
| --- | --- | --- |
| gitops `009-full-platform-rollout` | 36 | T038, T082, T092, T099–T117, T142, T145, T148–T153, T155, T157, T159–T162 |
| gitops `003-reusable-cicd-delivery` | 6 | T020–T021, T024, T029, T035, T038 |
| gitops `007-advanced-testing` | 6 | T011–T013, T018, T026–T027 |

### 0.5 The order the work has to happen in

**Gate 0 — restore the economical platform in the new account.** Five items,
all Esteban's, and nothing else in the project runs until they land:

1. gitops `009` T164 — commit the 66-file account repoint currently uncommitted.
2. The five service `ci.yml` files — see §0.7, this is the release-path unblock.
3. gitops `009` T165 — publish the five images through the OIDC release path
   into the replacement ECR repositories and promote their exact digests.
4. ops `001` T060 and gitops `009` T166 — merge through protected `main`, run
   `scripts/managed/bootstrap-cluster.sh` for `microtodosuite-dev`, and verify
   ArgoCD, External Secrets, workloads, and cross-service behavior read-only.

Everything in gitops `005`, `006` and `008` that is still open is live
observation on that cluster. Gate 0 is what unblocks Santiago, not Phase 4.

**Gate 1 — the Phase 4 apply chain** (gitops `009` T052–T067). Fifteen ordered
steps: shared egress, full dev, full prod, staging prerequisites, dev-owner
trust, GitOps bootstrap. Everything in US3, US4 and US5 depends on it, and
T061/T062 inside it is where the full-profile publisher trust is fixed.

**Gate 2 — the rest**, in each member's lane, with the Azure DR leg last.

### 0.6 What each member can do today, without waiting

**Santiago**, before the cluster returns:

- Resolve E1 first — spec `006` T004–T007 pin kube-prometheus v0.16.0, Grafana
  11.7.0, Loki 3.6.0 and Jaeger 1.65.0 against v0.18.0, v13.2.0, v3.7.6 and
  **v2.20.0** vendored. Jaeger is a major-version jump. That is a maintainer
  decision, not an edit, and everything else in `006` sits on top of it.
- Author, no cluster needed: `006` T014a saturation rules and T022 PromQL
  assertions; `008` T003–T004; `009` T084–T085, T087–T088, T090 manifests; and
  the two documentation debts, `006` T049 and `008` T027.

**Juan Manuel**, before Gate 1:

- `007` T011–T012 — the `frontend → auth-api` pact and the two provider
  verifications. One pact of three exists.
- `009` T082 mirror workflow, T099–T110 — matrix and workflow contract tests,
  the SHA pinning pass, canary render tests, and the fail-closed
  AnalysisTemplates.
- `009` T142, T145, T148–T149 — evidence fixtures, validator extension,
  read-only collection.
- `003-reusable-cicd-delivery` T020, T029.

### 0.7 Work no register covers

The five service repositories' `.github/workflows/ci.yml` each hard-code
`916491575487` in `ecr-repository` and `publisher-role-arn`
(auth-api, todos-api, users-api, frontend, log-message-processor — one file
each). gitops `009` T165 assumes the release path works; it does not name this
edit, and no other task does either.

Rule 5 of §7 of the conventions makes this need a task before it is done. The cheapest
correct fix is to extend T165's text to name the five workflow inputs, in the
same pull request that performs the repoint.

### 0.8 Not distributable

Seven acceptance tasks — gitops `009` T038, T067, T098, T117, T141, T152, T162
— are maintainer signatures, not work. They are counted above under whoever
owns the surrounding stage, but no member closes their own.

---

## 1. How even was the original split?

Measured by file touches across all nine repositories, attributed to the commit
author, and bucketed by the path each commit changed.

| Member | Area owned | INFRA | CI/CD | OBS | Total touches | Share of the active three |
| --- | --- | --- | --- | --- | --- | --- |
| Esteban Gaviria | Infrastructure | 399 | 136 | 3 | **538** | **65 %** |
| Santiago Valencia | Observability | 32 | 28 | 96 | **156** | **19 %** |
| Juan Manuel Díaz | CI/CD | 43 | 76 | 7 | **126** | **15 %** |

A fourth contributor, Juan David Colonia, has 101 touches (73 % CI/CD) confined
to 2025-04-20 → 2025-04-25 — the original Azure phase, before the migration
began. He does not appear in the current window and is excluded from the
forward allocation.

**The split was not even.** Two findings:

1. **Everyone stayed in their lane, and the lanes are the right shape.** Each
   member's largest bucket is the area they own: Esteban 74 % infrastructure,
   Santiago 61 % observability, Juan Manuel 60 % CI/CD. The assignment itself
   was coherent — nobody drifted.

2. **The lanes are very different sizes.** Esteban carried roughly 3.4× Santiago
   and 4.3× Juan Manuel. That is not a discipline problem; it is what the
   architecture demanded. Migrating from Azure Container Apps to a multi-cloud
   EKS/AKS estate is dominated by infrastructure, and the observability and
   CI/CD layers could not start until the foundation existed.

Left alone, the remaining work would repeat the imbalance: classified by
subject matter, the 236 outstanding tasks fall 137 infrastructure (58 %), 58
CI/CD (25 %), 41 observability (17 %).

## 2b. After the two decisions — the working numbers

> Superseded by §0.4. Retained for the sequencing argument below.

**192 tasks remain**, of which 24 are the deferred Azure leg.

| Member | Area | Near-term | Azure (last) | Total |
| --- | --- | --- | --- | --- |
| Esteban Gaviria | Infrastructure | **61** | 17 | 78 |
| Juan Manuel Díaz | CI/CD and delivery | **54** | 0 | 54 |
| Santiago Valencia | Observability | **53** | 7 | 60 |
| | | **168** | **24** | **192** |

Near-term is 36 % / 32 % / 31 %. That is as even as the plan admits, and it took
two decisions rather than any reshuffling of tasks.

### What "the Azure DR leg" actually is

009 T118-T141: an AKS 1.35 cluster in its own VNet with its own locked Azure Blob
state, a Key Vault seeded with exactly four secret names, an independent
in-cluster ArgoCD, the already-signed production digest mirrored to ACR without
rebuilding, a `full-prod-azure` DNS record, and then the chaos experiments and
the complete-AWS-outage game day that prove failover works. Route 53
latency-based active-active traffic is a further gated step (T140) needing its
own approval.

**Nothing of it exists yet** — there is no `ops/azure/` directory and no
`clusters/aks-dr/`. It has not been started early; it is correctly last both by
the recorded sequencing decision and by spec 009's own dependency graph, which
makes US5 depend on the production-validated US4 digest.

---

## 2c. The big changes each member still has to make

Not task lists — the shape of the work.

### Esteban — Infrastructure

1. **Run the Phase 4 apply chain (T052-T067).** Fifteen ordered steps: shared
   egress, full-dev, full-prod, staging prerequisites, dev-owner trust, GitOps
   bootstrap. This is the single unblocking action in the whole project.
2. **Fix the publisher OIDC trust (inside T061/T062).** Today the role refuses
   the service repositories' `main`, so no release or promotion PR can be
   produced at all.
3. **Finish the AWS dev foundation (ops-001, 23 tasks).** Backend tests, the
   `gitops_handoff` output contract, Infracost wiring, and a checks workflow that
   currently runs `terraform fmt` and nothing else.
4. **Stand up the infrastructure half of the platform**: Istio and Kiali (T083)
   — still entirely absent — plus Karpenter (T086), secret wiring (T089), and
   the activation waves (T091).
5. **Drive the live rollout** full-dev to staging to production (T093-T098).
6. **Last: the AKS foundation and DNS** (T118-T134).

### Juan Manuel — CI/CD and delivery

1. **Build the five-service quality matrix** and pin every action by full SHA
   (T099-T109). Every service workflow gains its complete blocking gate set.
2. **Implement progressive delivery**: the 10/25/50/100 canary with fail-closed
   p99 and error-rate AnalysisTemplates (T100, T110), then prove a deliberately
   unhealthy canary aborts and a rollback restores (T111-T117).
3. **Build the platform image mirror** (T082) — the OIDC workflow that copies
   upstream images to ECR. Nothing in Phase 5 can reference a digest until it
   runs.
4. **Close the contract-testing gap**: the `frontend -> auth-api` pact and the
   `auth-api <- frontend` and `users-api <- auth-api` provider verifications
   (007 T011/T012), plus the four observed-verification tasks.
5. **Harden the gates themselves**: path-scoped branch protection on production
   overlays (003 T020), and the extended evidence validator and stage-gate
   machinery (T142-T152).

### Santiago — Observability and monitoring

1. **Resolve the version drift first (E1).** Spec 006 pins four versions that
   were never vendored, including Jaeger 1.65.0 against a shipped v2.20.0 — a
   major version with a different architecture. Everything else in 006 sits on
   top of that unresolved question.
2. **Finish the observability foundation** (006, 23 tasks): saturation rules,
   alert firing and resolution evidence, trace retrieval, log search and
   trace-log correlation, and the Alertmanager routing commit.
3. **Replace the economical substitutions with the full stack**: ECK,
   Elasticsearch, Kibana, Logstash, and Filebeat (T084) in place of today's Loki
   and Alloy; and the full-profile Prometheus, Grafana, Jaeger, and OTel
   correlation (T085).
4. **Instrument the workloads**: probes, resource bounds, PodDisruptionBudgets,
   topology spread, ServiceMonitors, and KEDA ScaledObjects (T090), plus the
   frontend and users-api health and telemetry contracts (T073/T076/T078/T081)
   — the last two of five services still missing them.
5. **Add the remaining monitoring surfaces**: Chaos Mesh and OpenCost (T087),
   the Kyverno and Falco hardening (T088), continuous vulnerability assessment
   (T146), and the cost dashboard (T147).
6. **Last: run the DR game day** (T135-T141) and record continuity honestly —
   what was observed, lost, duplicated, or diverged.

### The dependency that matters more than the counts

Roughly half of Juan Manuel's and Santiago's lanes cannot start until Esteban's
apply chain lands. Both have real preparation work that does not need a live
cluster — vendoring components, writing tests, wiring workflows — and that is
what they should be doing while Phase 4 runs.

## 2d. Reassessment: counting tasks was the wrong measure

The 61 / 54 / 53 split above is **not** as even as it looks. Two corrections.

### Tasks are not the same size

Weighting each task by the artifacts it names — a rough but honest proxy, since
a task naming five components is five components of work — changes the picture:

| Member | Tasks | Share | Weight | Share | Weight per task |
| --- | --- | --- | --- | --- | --- |
| Esteban | 78 | 40 % | 436 | **51 %** | 5.6 |
| Juan Manuel | 54 | 28 % | 220 | 26 % | 4.1 |
| Santiago | 60 | 31 % | 202 | 23 % | 3.4 |

Counting tasks understates infrastructure by eleven points. Spec 009's Phase 5
bundles several components into one checkbox — T083 is "AWS Load Balancer
Controller **and** Istio **and** Kiali" in a single task, T084 is ECK plus
Elasticsearch, Kibana, Logstash, and Filebeat. A register with coarse tasks
looks lighter than it is.

### The real bottleneck is not the count, it is the bundling

Of the 78 tasks in the infrastructure lane, **44 actually require Esteban**:
live cloud reads, refreshed plans, state backups, exact-plan approval, bootstrap
mutations, merges through protected branches, and live verification. The other
**34 are authoring** — Terraform module code, mocked tests, contract checks,
vendored manifests, documentation — that need no cluster and no approval, and
could be written by anyone.

> **Corrected 2026-08-30.** This first read 28 / 50. That split was produced by
> a keyword pass that missed operational verbs — "execute exactly", "open five
> PRs", "run and fix", "verify each" — and so counted operating tasks as
> authoring. The twenty ambiguous cases were then judged individually. The
> corrected figure is 44 / 34, and it changes the conclusion's size though not
> its direction: there is real authoring to move off the critical path, but it
> is a third of the lane, not two-thirds.

That bundling is what makes infrastructure a single point of failure. Everyone
else is blocked behind the apply chain, and the person who has to run it is also
carrying fifty tasks of authoring that have nothing to do with it.

### What would actually be more equitable

1. **Split the infrastructure lane by kind, not by subject.** Esteban keeps the
   28 cloud-operation tasks — the apply chain, the live rollout, the bootstraps.
   The 50 authoring tasks are distributed by capacity, not by area. Most of
   ops-001's remainder (backend tests, the `gitops_handoff` contract, the checks
   workflow, Infracost wiring) is test-and-workflow work that sits closer to
   CI/CD anyway.
2. **Move component vendoring off the critical path.** T083 is manifest
   authoring; only its apply needs Esteban. The same holds for T086 and T089.
3. **Do not measure this in task counts again.** Measure it in weight, and check
   who is blocking whom.

Applying (1) and (2) leaves roughly: Esteban 44 cloud operations plus review of
everything that touches infrastructure; Juan Manuel and Santiago absorbing 34
authoring tasks across their existing lanes. That does not equalise the lanes —
infrastructure still carries the most, and most of what it carries genuinely
needs the infrastructure owner. What it does is stop one person's calendar from
gating work that never needed them in the first place.

**Recommendation**: the 61 / 54 / 53 split is defensible as a subject-matter map
and should not be used as a workload plan. Use §2d.

> **Not adopted.** The maintainer chose to keep the subject-matter lane whole;
> see §0.3. The measurement above still holds — it is the plan that changed.

---

## 2. The proposed allocation (pre-decision, retained for context)

Rebalanced by moving **whole coherent blocks**, never individual tasks, so each
boundary stays something one person can own without coordinating every day.

| Member | Area | Tasks | Share |
| --- | --- | --- | --- |
| Esteban Gaviria | Infrastructure | **78** | 33 % |
| Juan Manuel Díaz | CI/CD and delivery | **98** | 42 % |
| Santiago Valencia | Observability and monitoring | **60** | 25 % |

Three deliberate moves produce this from the 137/58/41 subject-matter split:

- **The local GitOps pilot (spec 001, 44 tasks) goes to CI/CD, not
  infrastructure.** What remains in it is a reproducible-run harness, evidence
  schemas, newcomer-workflow tests, and fixtures — delivery and quality work
  that happens to live in a cluster. Read its outstanding tasks and the shape is
  unmistakable.
- **The DR game day (009 T135–T141) goes to observability, not infrastructure.**
  Building AKS is infrastructure; proving failover behaviour, sampling
  continuity, and recording what was lost is monitoring.
- **The service health and telemetry contracts (009 T073, T076, T078, T081) go
  to observability.** They are probes, correlation IDs, and metric surfaces —
  the instrumentation layer, not application features.

### Infrastructure — Esteban Gaviria, 78 tasks

| Register | N | Tasks |
| --- | --- | --- |
| gitops `009-full-platform-rollout` | 49 | T052–T071, T083, T086, T089, T091, T093–T098, T118–T134, T154, T156 |
| ops `001-aws-dev-foundation` | 23 | T025–T028, T030–T031, T035–T036, T038–T050, T052–T053 |
| gitops `005-namespace-isolation` | 6 | T042–T043, T068, T075, T092, T095 |

The critical path lives here. T052–T067 is the ordered apply chain that every
other member is blocked behind; nothing in US3, US4, or US5 can start until it
lands.

### CI/CD and delivery — Juan Manuel Díaz, 98 tasks

| Register | N | Tasks |
| --- | --- | --- |
| gitops `001-local-gitops-pilot` | 44 | T002, T004–T015, T017–T019, T021, T026, T029–T047, T049–T053, T055–T056 |
| gitops `009-full-platform-rollout` | 42 | T038, T072, T074–T075, T077, T079–T080, T082, T092, T099–T117, T142, T145, T148–T153, T155, T157, T159–T162 |
| gitops `003-reusable-cicd-delivery` | 6 | T020–T021, T024, T029, T035, T038 |
| gitops `007-advanced-testing` | 6 | T011–T013, T018, T026–T027 |

Six of the 009 tasks (T072, T074, T075, T077, T079, T080) close when `gitops#78`
merges, so the real figure is 92.

### Observability and monitoring — Santiago Valencia, 60 tasks

| Register | N | Tasks |
| --- | --- | --- |
| gitops `006-observability-platform-foundation` | 23 | T003–T007, T009–T010, T014, T021–T022, T024–T026, T030–T031, T038–T039, T044–T049 |
| gitops `009-full-platform-rollout` | 21 | T073, T076, T078, T081, T084–T085, T087–T088, T090, T135–T141, T143–T144, T146–T147, T158 |
| gitops `005-namespace-isolation` | 8 | T064–T065, T069, T072, T079, T082, T089–T090 |
| gitops `008-security-runtime-hardening` | 8 | T003–T004, T013, T018, T023, T025–T027 |

Runtime security (Falco, kube-bench, kube-hunter, continuous vulnerability
assessment) and FinOps (OpenCost) sit here because they are monitoring surfaces.
If the team would rather treat security as its own concern, that is the obvious
place to split a fourth lane.

---

## 3. Not distributable

**Seven acceptance tasks** — 009 T038, T067, T098, T117, T141, T152, T162 — are
maintainer signatures, not work. They are counted in the tables above under
whoever owns the surrounding stage, but no member can close their own.

---

## 4. The two decisions that change these numbers most

Neither is a reallocation. Both are pending decisions recorded in
`plan-reconciliation.md`.

1. **Is the local GitOps pilot still a target?** Spec 001 stopped at 12/56 on
   2026-08-09 when work moved to the cloud, and nothing retires it. Its 44 tasks
   are the single largest block in the allocation. Retiring it takes CI/CD from
   98 to 54 and makes the split 78 / 54 / 60 — infrastructure heaviest again.

2. **Does the Azure DR leg stay in scope?** 009 T118–T141 is 24 tasks across
   infrastructure and observability. Deferring it takes infrastructure from 78
   to 61 and observability from 60 to 53.

Deciding both would leave roughly 61 / 54 / 53 — the most even the plan admits
without cutting into the full profile itself.

## 5. What is blocked, and on whom

Worth stating plainly, because an even task count is not an even schedule:

- **CI/CD and observability are both blocked behind infrastructure.** The Phase 4
  apply chain gates the platform components, the mirrored digests, and every
  live-evidence task. Until T052–T067 lands, roughly half of the other two lanes
  cannot start.
- **The publisher OIDC role rejects the service repositories' `main`**, so no
  release or promotion PR can be produced at all today. Its fix (T061/T062) is
  inside that same chain.
- **CVE-2026-14456 blocks todos-api and frontend CI** and needs a team decision,
  not an owner.

A fair reading is that infrastructure should be unblocked first even though its
count is not the largest, and that the other two lanes have preparation work
they can do now — vendoring components, writing tests, wiring workflows — that
does not need a live cluster.
