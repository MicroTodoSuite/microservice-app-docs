# Infrastructure Execution Plan

**Date**: 2026-09-09
**Owner**: Esteban Gaviria
**Scope**: the infrastructure lane of `full-platform/work-allocation.md` §0.4 —
**81 tasks**, after `gitops#86` ticked 009 T164.

| Register | N | Tasks |
| --- | --- | --- |
| gitops `009-full-platform-rollout` | 51 | T052–T071, T083, T086, T089, T091, T093–T098, T118–T134, T154, T156, T165–T166 |
| ops `001-aws-dev-foundation` | 24 | T025–T028, T030–T031, T035–T036, T038–T050, T052–T053, T060 |
| gitops `005-namespace-isolation` | 6 | T042–T043, T068, T075, T092, T095 |

This plan orders them, names what each block depends on, and states the three
things the registers do not say — one of which stops Phase 4 before it starts.

---

## 0. What is true today

Verified 2026-09-09 by read-only API calls against account `575172595729`, not
from the registers.

| Fact | Evidence |
| --- | --- |
| EKS `microtodosuite-dev` is ACTIVE, Kubernetes 1.35, created 2026-09-07 | `aws eks describe-cluster` |
| Its node group is ACTIVE: 2 × `m7i-flex.large`, On-Demand | `aws eks describe-nodegroup` |
| Ten ECR repositories exist, all `IMMUTABLE` | `aws ecr describe-repositories` |
| **All of them are empty** — zero images in all five neutral repositories | `aws ecr list-images` |
| The five economical overlays name digests that exist only in the retired registry | `apps/*/profiles/economical/overlays/*/kustomization.yaml` vs `aws ecr describe-images` → `imageDetails: []` |
| **The publisher role already trusts all five service repositories' `main`** | `aws iam get-role microtodosuite-github-ecr-publisher` |
| Cluster access entries exist only for `microtodosuite-terraform-dev` and the node role | `aws eks list-access-entries` |
| The local kubeconfig still points at the retired cluster | `kubectl config get-contexts` |

Two consequences worth stating plainly:

1. **E2 is fixed.** The reconciliation recorded the publisher OIDC role as
   rejecting the service repositories' `main`. The role rebuilt in the new
   account names all five subjects. What still fails is the *caller*: the five
   service `ci.yml` files ask for a role ARN in the retired account.
2. **Bootstrapping ArgoCD today would produce five `ImagePullBackOff`
   workloads.** The overlays point at the new registry with old digests, and the
   registry is empty. Publish first, promote second, bootstrap third — which is
   the order T165 and T166 already have.

Open pull requests: `ops#32` (needs no approval), `gitops#86`, `gitops#85`,
`gitops#84` (each needs one).

---

## 1. Three things the registers do not say

### 1.1 Spec 009 is still written against the retired account

The full-profile specification names `916491575487` in its specification,
plan, quickstart, data model, stage-gate contract, release-promotion contract,
research notes, and — decisively — in
`contracts/full-profile-evidence.schema.json` as
`"awsAccountId": { "const": "916491575487" }`. Every evidence bundle Phase 4
produces would fail its own schema.

Four Terraform roots hard-pin it too, in three places each:

| Root | `variables.tf` validation | `.tfvars` | `.s3.tfbackend` |
| --- | --- | --- | --- |
| `aws/shared/egress` | yes | — | bucket + KMS key |
| `aws/environments/full-dev/foundation` | yes | — | bucket + KMS key |
| `aws/environments/full-prod/foundation` | yes | — | bucket + KMS key |
| `aws/environments/demo-full/foundation` | — | yes | bucket + KMS key |

Those backends name a bucket and a KMS key in an account nobody can reach, so
`terraform init` fails before any plan is produced.

There is a second-order problem. Spec 009 T040 and T041 are **ticked**, and what
they delivered is "full-dev root tests for account `916491575487`" and the same
for full-prod. Those tests now assert the wrong account. Under §7 rule 8 of the
conventions this is a reconciliation, not an edit: the tasks are un-ticked and
re-delivered against the new account, or the discrepancy is recorded.

**No task covers any of this.** It is the first thing to write.

### 1.2 `demo-full` did not survive the account change

009 T056 says to *re-plan unchanged* `demo-full` and require exactly
`0 to add, 0 to change, 0 to destroy` before enabling the staging
prerequisites. That environment's state lived in the retired account's bucket
and its resources lived in the retired account. There is nothing to re-plan.

The last external backup is
`~/backups-microtodosuite/demo-full-foundation-pre-university-cidr-20260824T175510Z.tfstate.json`
— a record of what existed, not something that can be adopted into a different
account.

`demo-full` has to be created from scratch, which makes T056 a create-plan with
non-zero adds, and its acceptance text is wrong as written. **A maintainer
decision, not an edit.**

### 1.3 The full profile does not fit in this account's quotas

Measured 2026-09-09 in `us-east-1`:

| Quota | Limit | In use | Free |
| --- | --- | --- | --- |
| Elastic IPs | 5 | 3 | 2 |
| Standard On-Demand vCPUs | 16 | 4 | 12 |
| VPCs per region | 5 | 2 | 3 |
| Standard Spot vCPUs | 32 | 0 | 32 |
| EKS clusters | 100 | 1 | 99 |

What the full profile needs on top of that: a VPC for the shared egress hub
(`10.50.0.0/16`, with its own NAT and EIP), and VPCs for `full-dev`,
`full-prod` and `demo-full` — **four more VPCs against three free**. Each full
cluster runs at least two 2-vCPU nodes, so **twelve more vCPUs against twelve
free**, leaving nothing for a rolling replacement. `demo-full` carries its own
NAT, so **two more EIPs against two free**, leaving nothing.

Every axis lands exactly at or over the ceiling. The design is already doing
the right thing — full-dev and full-prod route through the Transit Gateway
instead of running their own NAT, which is why the EIP count is merely tight
rather than impossible.

Three ways out, in increasing order of cost to the project:

1. **Delete the default VPC** (`172.31.0.0/16`, unused). Frees exactly the one
   VPC the plan is short. Cheapest, reversible, and it should happen anyway.
2. **Request quota increases.** Days of lead time, and a young account is a
   plausible refusal. Start it early if it is going to happen at all.
3. **Reduce the economical dev from three NAT gateways to one.** Frees two EIPs
   and roughly USD 64 a month. It also changes the live platform that is the
   rollback target for the whole rollout, so it is a decision, not a cleanup.

009 T052 exists to discover exactly this and "stop on any mismatch". Run it
before anything else in Phase 4 — the answer is already known to be tight, and
it is better to find the wall from a quota call than from a half-applied
environment.

---

## 2. The plan

Eight blocks. Every task in the lane appears in exactly one.

| Block | What it is | Tasks | Needs a live cluster? |
| --- | --- | --- | --- |
| A | Restore the economical platform | 3 | it *creates* it |
| B | Foundation debt, no cloud needed | 21 | no |
| C | Close spec 005 and the two US1 leftovers | 8 | yes |
| D | Re-point spec 009 at the new account | 0 registered | no |
| E | The Phase 4 apply chain | 16 | yes |
| F | Platform components and the live rollout | 14 | partly |
| G | Azure disaster recovery | 17 | yes, on Azure |
| H | Final closure | 2 | yes |
| | | **81** | |

### Block A — Restore the economical platform

The entire project is stopped behind this. Four steps, in order:

1. **Merge what is already written.** `ops#32` needs no approval. `gitops#86`,
   `#85` and `#84` each need one reviewer — ask for it today; they have been
   open for days and their checks are green.
2. **Repoint the five service workflows.** `.github/workflows/ci.yml` in
   auth-api, todos-api, users-api, frontend and log-message-processor each name
   the retired account in `ecr-repository` and `publisher-role-arn`. The role
   they need already exists and already trusts them. Five one-line-pair pull
   requests; no Terraform, no gate. **This is the single cheapest unblock in the
   project.**
3. **009 T165** — let the release path publish the five images into the
   replacement repositories, then promote their exact digests into the
   economical overlays.
4. **ops 001 T060 and 009 T166** — merge the reviewed GitOps revision through
   protected `main`, run exactly `scripts/managed/bootstrap-cluster.sh` for
   `microtodosuite-dev`, and verify ArgoCD, platform Applications, External
   Secrets, workloads and cross-service behaviour read-only.

Before step 4, refresh the kubeconfig for the new cluster through
`microtodosuite-terraform-dev`; the IAM user has no access entry and never will.

**Done when** five services are Healthy and Synced from digests that exist in
`575172595729`. That single event unblocks Santiago's whole lane (specs 005, 006
and 008 are live observation on *this* cluster) and Juan Manuel's promotion work.

Tasks: gitops 009 T165, T166 · ops 001 T060. **Size: M** — mostly waiting on
pipelines, plus one bootstrap that must be exactly two mutations.

### Block B — Foundation debt that needs no cloud

Twenty-one ops-001 tasks, none of which need a cluster, an approval, or a plan.
This is the work to do while Block A's pipelines run and while decisions are
pending.

| Group | Tasks | What it is |
| --- | --- | --- |
| State backend | T025–T028, T030–T031, T035–T036, T038 | Mocked `tftest.hcl` assertions, prefix-scoped IAM, the missing backend-policy ARN output, the bootstrap README, an opt-in two-writer lock check |
| `gitops_handoff` | T039–T044 | The typed non-sensitive handoff object, its schema tests, the sibling contract check, module exports, the mapping README |
| Repository hygiene | T045–T050 | Infracost config, the checks workflow that today runs `terraform fmt` and nothing else, the root README, the full local check run, the four-command preview, the quickstart reconciliation |

Two of these are already recorded as partial in the reconciliation: T031 (the
outputs lack the approved backend policy ARN) and T046 (the workflow runs
formatting only). Neither needs anything but time.

T049 says "in an approved account with an already conforming backend" — that is
now true for the first time in weeks, so it can be run against the new account.

**Size: L in aggregate, S each.** Nothing here blocks anyone.

### Block C — Close spec 005 and the two US1 leftovers

Entry condition: Block A is done.

- gitops 005 **T042–T043** — open the five short-lived service pull requests,
  obtain review and green checks, then observe five reviewed `main` publication
  runs and record tests, Trivy, SBOM, digest and keyless signature identity.
  Block A step 2 is what makes these possible.
- gitops 005 **T068, T075, T092, T095** — the activation pull request and its
  exact SHA, the continuity chain through final cleanup, the final evidence PR,
  and the post-approval recovery step.
- ops 001 **T052** — a refresh-backed no-apply plan proving only the expected
  in-place VPC CNI update, once the execution role has the documented refresh
  permissions.
- ops 001 **T053** — read-only confirmation that every ready `aws-node` pod has a
  ready `aws-eks-nodeagent` with `--enable-network-policy=true`.

**Size: M.** Mostly observation and evidence capture, which is slow but not hard.

### Block D — Re-point spec 009 at the new account

**This block has no tasks yet. Write them first.** §1.1 is its content:

1. A recorded decision amending spec 009's account across `spec.md`, `plan.md`,
   `quickstart.md`, `data-model.md`, `contracts/stage-gate-contract.md`,
   `contracts/release-promotion-contract.md`,
   `contracts/full-profile-evidence.schema.json` and `research.md`.
2. Un-tick and re-deliver 009 T040 and T041, whose tests assert the retired
   account.
3. Repoint four Terraform roots — `variables.tf` validation, `.tfvars`, and the
   three `.s3.tfbackend` files, which name a bucket and KMS key that no longer
   resolve.
4. Repoint the full-profile overlays in `apps/*/profiles/full/overlays/`.
5. Decide `demo-full`: re-create it, or descope staging from the rollout. If it
   is re-created, T056's acceptance text has to change with it.

Do this as its own specification change, not inside a feature branch, and give
it task IDs so the register carries it. Everything in Block E depends on it.

**Size: M**, and it is the highest-leverage writing in the plan.

### Block E — The Phase 4 apply chain

Sixteen ordered tasks, gitops 009 T052–T067. Entry condition: Block D merged and
the quota question answered.

| Step | Tasks | Gate |
| --- | --- | --- |
| Discovery | T052 | Stops on any quota, collision or ownership mismatch |
| Saved plans | T053, T054, T055, T056 | Egress, full-dev, full-prod, staging prerequisites — each with Infracost and a no-destroy proof |
| Applies | T057, T058, T059, T060 | Each needs exact-plan approval **and** a timestamped external state backup under `~/backups-microtodosuite/` |
| Dev-owner trust | T061, T062 | One new `microtodosuite/platform` repository, mirror role, issuer trust updates, zero change to the five service repositories |
| GitOps roots | T063, T064, T065 | Three empty full-cluster roots merged through protected `main`, then exactly two audited bootstrap mutations per cluster, then read-only verification |
| Isolation proof | T066 | NAT/TGW/IGW routes and flow logs prove no dev-to-prod path |
| Acceptance | T067 | Maintainer signature against SC-002, SC-003, SC-012–SC-014 |

Rules that hold for every apply in this block, from `AGENTS.md` and the
constitution: only an approved saved plan, never a convenience apply; a
timestamped external backup first, or a `no-prior-state` receipt; and after any
change under `aws/`, the refreshed economical dev plan must still report
`0 to add, 0 to change, 0 to destroy`, verified from `terraform show -json` with
every `resource_changes` entry `no-op` — not from the printed text.

**Size: L, and it is the critical path.** Half of both teammates' lanes sit
behind it.

### Block F — Platform components and the live rollout

Fourteen tasks in two halves.

**Authoring, no cluster needed — start during Block A or B.** 009 T068–T071 are
four failing test suites: the capability/version/resource-budget/image inventory,
mesh and network render tests, cloud-secret tests, and platform behaviour tests.
They gate T083–T091 by design, so writing them early costs nothing and buys
schedule.

**Vendoring and activation**: T083 (AWS Load Balancer Controller 3.5.0, Istio
1.30.3, Kiali 2.31.0), T086 (Karpenter 1.14.1 with Spot-only NodePools under a
24-vCPU ceiling), T089 (cloud SecretStore/ExternalSecret overlays), T091 (the
dependency-wave activation).

> **Cross-lane dependency.** T083 and T086 must reference *mirrored ECR
> digests*, and the mirror workflow is 009 T082 — Juan Manuel's task. Nothing in
> this half can be finished until his mirror runs. Tell him now; it is the one
> place where the infrastructure lane is blocked on someone else.

**Live rollout**: T093–T098 — merge the full-dev dependency waves through the
Istio NLB, implement the Route 53 records, activate one signed service and then
all five, promote to staging and to AWS production with shared traffic still
disabled, and validate the evidence bundle.

**Size: L.**

### Block G — Azure disaster recovery

Seventeen tasks, 009 T118–T134, last by recorded sequencing decision and by
spec 009's own dependency graph — US5 depends on a production-validated US4
digest.

Nothing of it exists: no `ops/azure/`, no `clusters/aks-dr/`. T124 is the
entry point — install the checksum-pinned Azure CLI and run the read-only
`scripts/preflight/azure-dr.sh` discovery that is already written.

**Prerequisite nobody has confirmed**: an active Azure subscription. The
Container Apps estate was retired partly because its credential had expired. If
there is no subscription, this block is not "last", it is "not possible", and
that changes the shape of the whole rollout. Answer this before Block E, not
after Block F — it is one command and it may save seventeen tasks of planning.

**Size: XL**, and it is the block most likely to be descoped.

### Block H — Final closure

009 T154 (Terraform fmt/validate/test and refreshed drift plans across dev,
demo-full, full-dev, full-prod, shared egress and Azure DR, requiring zero
unintended changes) and T156 (read-only live acceptance across every
destination). Both are aggregate checks; they close when everything else does.

---

## 3. Order

```
now ──► A (restore) ──► C (spec 005, US1 leftovers)
         │
         ├─► B (ops-001 debt) ......... any time, no dependencies
         ├─► F-authoring (T068–T071) .. any time, gates F-vendoring
         └─► D (re-point 009) ──► E (Phase 4) ──► F-vendoring ──► F-rollout ──► G ──► H
                                        ▲
                                        └── quota answer (§1.3) required here
```

Three things run in parallel with everything: Block B, the four test suites in
Block F, and the quota decision. Everything else is a chain.

The two long poles are the quota lead time — if increases are requested, days
before Block E can start — and Block E itself, which is fifteen ordered applies
each needing a backup and an approval.

## 4. Decisions needed, and when

> **Updated 2026-09-10** in `full-platform/capacity-and-regions.md`: decision 4 is
> answered (an Azure for Students subscription, six vCPUs per region); decision 3
> is replaced by the limits L1–L6, which fit both profiles at once by putting
> `full-prod` in a second region; and two decisions are added — the region layout
> (6) and economical cold DR (7). §5's recommendation is implemented: the account
> is declared once per repository and moved with `scripts/set-aws-account.sh`.

| # | Decision | Needed before | Owner |
| --- | --- | --- | --- |
| 1 | Amend spec 009's account, and how to handle the two false ticks (T040/T041) | Block D | Maintainer |
| 2 | Re-create `demo-full` or descope staging | Block D | Maintainer |
| 3 | Quotas: delete the default VPC, request increases, or reduce the economical dev to one NAT | Block E | Maintainer |
| 4 | Is there an active Azure subscription? | Block G, but ask now | Maintainer |
| 5 | Spec 006's version drift (E1) | Blocks Santiago, not this lane | Maintainer |

## 5. One recommendation that is not a task yet

**Make the AWS account a parameter.**

The project has now lived in three accounts: `995253610162`, then
`916491575487`, then `575172595729`. Each move costs a repository-wide sweep —
this one was 66 files in `gitops` alone, and it is still not finished, because
spec 009 and four Terraform roots were missed.

The account is hard-coded in `variables.tf` validation strings, in `.tfvars`, in
three `.s3.tfbackend` files, in specification prose, in a JSON schema `const`,
in test fixtures, and in five service workflows in five other repositories.

Concretely: one non-secret account input resolved in one place per repository, a
schema that takes a pattern instead of a `const`, a `scripts/repoint-account.sh`
with a contract test that fails when a retired account ID appears anywhere
outside `evidence/`, and the retired IDs listed as forbidden. That turns the
fourth migration into a variable change and a test run.

It belongs in ops `001` as a task. Given the evidence — twice in three weeks —
it will pay for itself.
