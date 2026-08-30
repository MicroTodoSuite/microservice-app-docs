# Work Allocation

**Date**: 2026-08-30
**Basis**: the 236 tasks outstanding once `gitops#80` lands, classified from
`full-platform/plan-reconciliation.md`.

The team split the project three ways at the start — infrastructure, CI/CD,
observability and monitoring. This document answers two questions with data
rather than impression: how even that split turned out, and how the remaining
work divides so each member has a boundary they can act on alone.

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

---

## 2. The proposed allocation

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
