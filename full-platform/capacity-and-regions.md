# Capacity, Regions, and the Account Parameter

**Date**: 2026-09-10
**Status**: proposal. Every item marked *Decision* is the maintainer's to take.
**Relates to**: `full-platform/infrastructure-execution-plan.md` §1 and §4.

This document answers four questions raised after the execution plan:

1. What exactly is decision 1, and what does taking it produce?
2. How is `demo-full` recreated in the new account?
3. What limits let the economical and the full profile run **at the same time
   in the same account**?
4. Did the design separate regions — and if `us-east-1` fails, what fails?

It does **not** decide which resources are persistent and which are ephemeral.
That is separate work in progress, and nothing here should be read as
prescribing it. This document fixes only the peak: what must fit when
everything is up at once.

---

## 1. Decision 1 — re-point spec 009, and do it once

### What is pinned

Spec 009 was written when the account was `916491575487`. Its Assumptions state
it as an approved value: *"AWS account `916491575487` and region `us-east-1`
remain the approved AWS…"* (`spec.md`). The same value is repeated in `plan.md`,
`quickstart.md`, `research.md`, `data-model.md`, both contracts, and the
evidence schema, where it is `"awsAccountId": { "const": "916491575487" }` — so
every Phase 4 evidence bundle would fail its own validation.

It is also in code: the `variables.tf` validation of `aws/shared/egress`,
`full-dev` and `full-prod` accepts only that account, and their Terraform tests
assert it. `config/aws-account-exceptions.txt` in both `ops` and `gitops` now
lists every such file, with the account it still carries and why.

### Why it is a decision, not a sweep

Only four documents may modify the plan, and spec 009's Assumptions are one of
them. An approved value in a specification changes by amendment. There is also a
register consequence: **spec 009 T040 and T041 are ticked**, and their text is
literally *"Add full-dev root tests for account `916491575487`, `us-east-1`,
`10.40.0.0/16`…"* (T041: `10.30.0.0/16`). The artifact exists and asserts the
wrong account. §7 rule 8 of the conventions forbids editing either side quietly.

### The two ways to take it

| | A — account only | B — account and region, together |
| --- | --- | --- |
| Spec 009 text | Replace the literal account with "the account declared in `config/aws-account.env`"; schema `const` becomes a 12-digit pattern checked against the declaration | The same, plus the region topology in §4 of this document |
| Terraform | Remove the account hard pin from three roots; they read the declaration like `dev` already does | The same, plus `full-prod` moves to `us-east-2` with its own NAT and leaves the egress hub |
| Register | Re-deliver T040 and T041 against the declared account | Re-deliver T040 and T041; amend T055 ("no NAT/EIP" no longer holds for prod) |
| Cost | One amendment now, a second one if regions change | One amendment |

**Recommendation: B.** Both changes rewrite the same Assumptions, the same
`full-prod` validation block (which pins the account *and* `us-east-1`), and
the same T041 test. Doing A and then B re-delivers T041 twice.

Either way, the amendment should reference the declared account rather than name
one. That is what makes this the last time spec 009 has to change for an
account move.

### What taking it produces

1. A spec 009 amendment in `gitops`, with task IDs for the re-point (Block D of
   the execution plan).
2. An `ops` change removing the three hard pins and re-delivering the root tests
   under the SDD pair — which removes eight entries from
   `config/aws-account-exceptions.txt`.
3. A `gitops` change re-pointing the fifteen full-profile overlays, the platform
   mirror in `full-profile-toolchain.lock`, the evidence collector, and the five
   evidence fixtures — which removes twenty-three more.

When the exception lists are empty except for the three that are not account
configuration at all, decision 1 is done, and the contract proves it.

---

## 2. Recreating `demo-full`

`demo-full` is the full-profile staging cluster (`spec.md` FR-007). Its state
lived in the retired account's bucket; nothing survives to adopt. It is a
**create**, from a new state key.

### Choose its shape first

The egress hub already reserves a route table for it —
`aws/shared/egress/variables.tf` lists `full-staging = 10.20.0.0/16` among its
spokes — but the `demo-full` root has no transit-gateway input and still builds
its own NAT. Recreating it is the cheapest moment to pick:

| | Own NAT (as before) | Transit spoke (as the hub expects) |
| --- | --- | --- |
| Can start | Now | After the egress hub is applied (T053, T057) |
| Elastic IPs | 1 | 0 |
| Code change | None | Add the transit input, as `full-dev` has |
| Later cost | A second plan and apply to convert | None |

*Decision 2*: recreate now with its own NAT, or wait for the hub and recreate it
once as a spoke. §3's limits assume the spoke.

### Runbook

Everything below is a human operation. It follows `AGENTS.md`: a saved plan
only, a state receipt first, an approval of the exact plan, and the economical
dev plan still `0/0/0` afterwards.

1. **Register it first.** No task covers recreating `demo-full` in the new
   account; ops spec 002 is 22/22. Add the tasks before the work, by §7 rule 5.
2. **Authenticate** as `microtodosuite-terraform-dev` in `575172595729`. The CLI
   session was expired on 2026-09-10.
3. **Update the local, gitignored inputs.** `demo-full.tfvars` still names the
   retired account — `scripts/set-aws-account.sh` reports it and deliberately does
   not edit it. Change `expected_account_id` and `bootstrap_admin_principal_arns`;
   keep the four `/32` operator CIDRs, which never enter Git.
4. **Write the backend file.** `demo-full.s3.tfbackend` is `dev.s3.tfbackend`
   with `key = "environments/demo-full/foundation/terraform.tfstate"`. The old
   local copy names the retired bucket and KMS key.
5. **Initialize**:
   `terraform -chdir=aws/environments/demo-full/foundation init -reconfigure -backend-config=demo-full.s3.tfbackend`
6. **Prove the key is new** — `aws s3api head-object` on it returns 404 — and
   write a `no-prior-state` receipt under `~/backups-microtodosuite/`.
7. **Plan and inspect**: `terraform plan -var-file=demo-full.tfvars -out=<ts>.tfplan`,
   then `terraform show -json <ts>.tfplan`. Require zero `delete` actions, no
   shared resource created (`create_shared_resources = false`), and the caller in
   `575172595729`. Run Infracost on the same plan.
8. **Approve that exact plan**, then `terraform apply -input=false <ts>.tfplan`.
9. **Back up the resulting state** externally, as for the dev foundation.
10. **Re-plan the economical dev** and confirm every `resource_changes` entry is
    `no-op` from the plan JSON.

The GitOps side waits: `clusters/eks-full-staging/` stays empty until spec 009
T063–T065, so recreating the cluster activates nothing.

---

## 3. Limits that let both profiles run at once

### What the account allows

Measured in `us-east-1` on 2026-09-09:

| Quota | Limit | Used by the economical profile |
| --- | --- | --- |
| VPCs | 5 | 2 (`dev`, plus the unused default VPC) |
| Elastic IPs | 5 | 3 (one NAT per AZ) |
| Standard On-Demand vCPUs | 16 | 4 (2 × `m7i-flex.large`) |
| Standard Spot vCPUs | 32 | 0 |

Every one of those quotas is **per region**. That is the lever: the second
region in §4 is not only a resilience decision, it doubles every ceiling.

### Proposed limits

| # | Limit | Why |
| --- | --- | --- |
| L1 | Delete the default VPC in `us-east-1` | Unused; frees the one VPC that was short |
| L2 | `demo-full` is a transit spoke, no own NAT | The hub already expects it; saves an Elastic IP |
| L3 | Bootstrap node groups: economical 2, every full cluster **1** | `full-dev` and `full-prod` already use one node and let Karpenter add the rest; `demo-full` drops from 2 to 1 |
| L4 | Karpenter Spot ceiling **8 vCPU per full cluster** | 8 + 8 + 8 = 24, exactly the aggregate T086 already specifies |
| L5 | `full-prod` in `us-east-2`, its own single NAT | §4; it cannot use a hub in another region |
| L6 | The economical profile keeps its three NATs | It is the live rollback target; one NAT would save about USD 64 a month and two Elastic IPs, but it is not needed to fit |

### The result

| Region | VPCs | Elastic IPs | NAT gateways | On-Demand vCPU | Spot vCPU ceiling | EKS clusters |
| --- | --- | --- | --- | --- | --- | --- |
| `us-east-1` | 4 / 5 | 4 / 5 | 4 | 8 / 16 | 16 / 32 | 3 |
| `us-east-2` | 1 / 5 | 1 / 5 | 1 | 2 / 16 | 8 / 32 | 1 |

`us-east-1` holds the economical cluster, `full-dev`, `demo-full` and the egress
hub; `us-east-2` holds `full-prod`. Every axis keeps headroom, and the
On-Demand headroom matters most: a rolling node replacement briefly doubles a
node group.

**Not yet verified**: that `us-east-2` in this account has the same default
quotas, and that it offers `m7i-flex.large`. New accounts do not always get
identical regional defaults. Check both before deciding:

```bash
aws service-quotas get-service-quota --region us-east-2 --service-code ec2 --quota-code L-1216C47A
aws service-quotas get-service-quota --region us-east-2 --service-code ec2 --quota-code L-0263D0A3
aws service-quotas get-service-quota --region us-east-2 --service-code vpc --quota-code L-F678F1CE
aws ec2 describe-instance-type-offerings --region us-east-2 --location-type availability-zone \
  --filters Name=instance-type,Values=m7i-flex.large
```

### What the peak costs

Approximate on-demand list prices, excluding data transfer, EBS, logs and Spot;
Infracost on the real plans is the number that counts:

| Item | Count | USD per hour |
| --- | --- | --- |
| EKS control planes | 4 | 0.40 |
| NAT gateways | 5 | 0.23 |
| Transit gateway attachments | 3 | 0.15 |
| On-Demand bootstrap nodes | 5 | 0.48 |
| Public IPv4 addresses | 5 | 0.03 |
| **Everything up** | | **about 1.28** |

About USD 31 a day, or about USD 930 for a month left running. How many hours a
month it actually runs is exactly what the persistent/ephemeral work decides.

---

## 4. Regions — what the design does today, and what fails with `us-east-1`

### What it does today

Nothing in AWS is separated by region. Every Terraform root targets `us-east-1`,
and `full-dev` and `full-prod` *require* it in their validation, because a
transit-gateway attachment is regional. The only other region in `ops` is a test
proving a wrong region is rejected.

The original plan's regional resilience comes from the other cloud: AKS in Azure
is the production disaster-recovery target (evolution plan §13, constitution
principle 12). The economical profile deliberately has none — its plan column
reads *"Resilience through multiple AZs within the same cluster; optional cold DR
via Velero"*.

The account-level singletons are regional too, and all of them are in
`us-east-1`: the state bucket and its KMS key, the ECR repositories, and the
Secrets Manager secrets. IAM, the GitHub OIDC provider and Route 53 are global.

### What a `us-east-1` outage takes down today

Everything that exists: the economical platform — the only live one — every
full-profile AWS environment as designed, every image, every secret, and the
ability to change anything with Terraform. The AKS cluster would survive only if
its images were already mirrored to ACR and its secrets already seeded, and both
of those workflows run *from* `us-east-1` (spec 009 T131, T133).

So yes: as designed, one region failing takes down everything on AWS.

### Proposed separation

| Where | What | Why |
| --- | --- | --- |
| AWS `us-east-1` | Economical platform, `full-dev`, `demo-full`, egress hub | Non-production and the live rollback target; the hub stays useful because its spokes share a region |
| AWS `us-east-2` | `full-prod` | The only environment whose outage users see; an independent region at `us-east-1` prices, closer to Colombia than the west coast |
| Azure | AKS disaster recovery for production | A different provider, as planned |

`us-west-2` is the alternative for `full-prod` if geographic distance matters
more than latency.

For production to actually survive a `us-east-1` failure, three regional
singletons need a copy in `us-east-2`:

- **ECR** — an account-level replication rule to `us-east-2`, so prod pulls from
  its own region;
- **Secrets Manager** — replica secrets in `us-east-2` for the production names;
- **State** — S3 replication of the state bucket, or an explicit acceptance that
  during a `us-east-1` outage prod keeps running but cannot be changed with
  Terraform until the region returns.

*Decision 6*: the region layout, and which of those three copies to build.

The economical platform cannot move: spec 009 forbids repurposing or destroying
it. What it can have is the cold DR its own plan column already names — Velero
backups to a bucket in `us-east-2`, restorable into a new cluster there. That is
reconciliation Part C item 3, still undefined. *Decision 7*: build it, or accept
that the economical profile is single-region by design.

### The Azure leg

Checked read-only on 2026-09-10:

- The subscription is **Azure for Students**, state `Enabled`. Decision 4 is
  answered: Block G is possible.
- **Six regional vCPUs**, in every region checked (`eastus2`, `centralus`,
  `westus2`, `brazilsouth`); the B-series family has 4 and DSv5 has 0. The DR
  cluster — ArgoCD, External Secrets, five services, and a mesh if it keeps one —
  has to fit in six vCPUs. Spec 009 T125's sizing has to be checked against that
  before it is written, not after.
- A student subscription runs on credit. When the credit is exhausted, its
  resources stop. A DR target that disappears when a budget runs out needs a
  budget alert and an owner.
- **Pick a region away from Northern Virginia.** Azure East US and East US 2 are
  in Virginia, the same broad geography as AWS `us-east-1`. Central US or South
  Central US gives the DR leg a genuinely different failure domain.
- The installed Azure CLI is 2.90.0; `scripts/managed/full-profile-toolchain.lock`
  pins 2.89.1 by checksum, and T124 names the pinned artifact. Install the pinned
  package, or bump the lock through a reviewed change.

---

## 5. The account parameter — how to change the account now

The account is declared once per repository that has to name it, and read from
there:

| Where | Declaration | Enforced by |
| --- | --- | --- |
| `microservice-app-ops` | `config/aws-account.env` | `tests/contract/aws-account-parameter.sh`, `tests/contract/set-aws-account.sh` |
| `microservice-app-gitops` | `config/aws-account.env` | the same two contracts, in `validate-gitops` |
| Five services and the reusable workflow | organization variable `AWS_ACCOUNT_ID` | each repository's CI contract |

Moving to another account is then:

```bash
scripts/set-aws-account.sh <new-account-id>     # in microservice-app-ops
scripts/set-aws-account.sh <new-account-id>     # in microservice-app-gitops
gh variable set AWS_ACCOUNT_ID --org MicroTodoSuite --body <new-account-id>
```

The script refuses a malformed, placeholder, current, or retired account; never
rewrites specifications or evidence; names the gitignored inputs that still
need a hand edit; and ends by running the contract. It changes files only —
the new account's backend and foundation are still applied from reviewed saved
plans.

GitOps manifests keep the account as a literal on purpose. ArgoCD renders plain
Git, IAM policies take literal ARNs, and an image reference is a literal string.
What changed is that the literal now has one declared source, one command to
move it, and a contract that fails when any copy disagrees.

---

## 6. Decisions, updated

| # | Decision | Status |
| --- | --- | --- |
| 1 | Re-point spec 009 — A (account only) or B (account and region) | Open; B recommended |
| 2 | Recreate `demo-full` now with its own NAT, or later as a transit spoke | Open |
| 3 | The limits L1–L6 | Open; replaces the three quota options in the execution plan |
| 4 | Is there an Azure subscription? | **Answered**: Azure for Students, six vCPUs per region |
| 5 | Spec 006's version drift (E1) | Open; unchanged, see the execution plan |
| 6 | Region layout, and which of ECR, Secrets Manager and state to copy to `us-east-2` | Open |
| 7 | Economical cold DR through Velero, or single-region by design | Open |
