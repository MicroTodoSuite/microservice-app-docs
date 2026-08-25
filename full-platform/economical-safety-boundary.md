# Economical Safety Boundary

The economical platform is the system that currently works. Throughout the
full-profile rollout it is the **rollback target**, never a cleanup target: no
full-profile stage may retire, degrade, or reclaim any part of it.

This document names who owns that boundary, what forces a stop, and which
actions are prohibited outright. It is governed by
`microservice-app-gitops/specs/009-full-platform-rollout/` and by SC-001: every
application and capability healthy before a stage is healthy after it, with zero
rollout-caused interruption and zero destructive state changes.

## What the boundary covers

| Asset | Owner | Authority |
| --- | --- | --- |
| Economical Kubernetes desired state (`profiles/economical/**`, `environments/**`) | GitOps maintainer | `microservice-app-gitops` |
| Economical cluster `eks-dev` and its ArgoCD registration | GitOps maintainer | Cluster + `clusters/eks-dev/**` |
| Dev foundation Terraform and its state | Infrastructure maintainer | `microservice-app-ops` |
| Account-level singletons: neutral ECR repositories, GitHub OIDC provider, shared reader roles, secret containers | Infrastructure maintainer | Dev foundation state |
| Legacy hosted zone `microtodosuite.abrdns.com` | Infrastructure maintainer | Dev foundation state |
| Release pipeline: CI, promotion, image signing | Organization automation maintainer | `.github` |

A full-profile stage may **add** alongside these. It may not rename, replace,
re-address, or delete any of them.

## Measured, not assumed

The boundary is enforced by evidence, not by intention:

- `scripts/managed/capture-economical-baseline.sh` records the pre-stage
  baseline — Applications, workloads, endpoints, namespace isolation, and the
  refreshed dev plan. It is strictly read-only and fails closed.
- `scripts/managed/evaluate-stage-gate.sh` decides whether a stage may start.
  **`accepted` is the only decision that unlocks a dependent.** `approved` does
  not: approval covers the reviewed inputs, while acceptance means the work ran
  and its evidence passed.
- The post-stage baseline must match the pre-stage one. An unreachable cluster
  or an Application synced to a different revision is missing evidence, never an
  implicit pass.

## Abort criteria

Stop the stage immediately, before any further action, when any of these is
true:

1. A refreshed economical Terraform plan is not clean, or the backend is
   inaccessible.
2. A plan shows an unexpected destroy, replacement, or re-addressing of any
   asset in the table above.
3. An economical ArgoCD Application leaves `Synced`/`Healthy`, or a workload,
   endpoint, or namespace isolation check regresses against the baseline.
4. A singleton would be duplicated — a second ECR repository, OIDC provider,
   shared role, or hosted zone for the same purpose.
5. A quota, cost, or capacity limit would be exceeded, or the exact cost was
   not accepted by the operator who owns it.
6. A DNS record or delegation could be lost, or the legacy zone would be
   renamed or destroyed.
7. Evidence is missing, stale, or unverifiable for any mandatory check.

On abort: mark the stage `blocked` in its evidence bundle with the exact failing
command and artifact checksum, notify the owner in the table above plus the
owner of the failed dependency, and use only the rollback the stage defined.

## Prohibited actions

These are prohibited during the rollout regardless of urgency or approval level:

- **Never** `kubectl apply|create|patch|delete|scale|replace` against a
  GitOps-managed cluster. Desired state changes go through a reviewed commit and
  ArgoCD reconciliation. The only exception is the two audited bootstrap
  mutations that register a brand-new cluster.
- **Never** apply a Terraform plan under an approval given for a different plan.
  An approval binds to the saved-plan checksum reviewed at that moment.
- **Never** run `terraform destroy`, `state rm`, `state mv`, `taint`, or
  `import` against economical state as part of a full-profile stage.
- **Never** rename `public_hosted_zone_name`. The name forces replacement, which
  destroys the zone, drops every record, and invalidates registrar delegation.
  The canonical domain exists at its own resource address.
- **Never** force-push, bypass branch protection, or merge with `--admin`.
- **Never** widen an IRSA trust to a wildcard subject, or extend the release
  publisher with cluster, mirror, or secret-read trust.
- **Never** treat a degraded economical platform as an acceptable cost of
  progress. The rollout waits; the platform does not.

## Rollback

The economical platform's rollback is a `git revert` of the reviewed commit
plus ArgoCD reconciliation — never a manual cluster edit. Infrastructure
rollback is a reviewed plan applied under its own approval, never an ad-hoc
state operation.

If a full-profile stage fails, the correct end state is the economical platform
exactly as the pre-stage baseline recorded it. Reaching that state is the
priority; diagnosing the failed stage comes after.
