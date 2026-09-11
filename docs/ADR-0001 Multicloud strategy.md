# ADR-0001 — Multicloud Strategy

**Date**: 2026-09-11
**Status**: Proposed. It changes constitution principle 12 and spec 009 user
story 5, so it takes effect only through the constitution amendment that
accompanies it.
**Decision owner**: the maintainer.

## Context — why two clouds were chosen

The reason is written down in two places:

- The evolution plan, §13: AWS EKS is primary and Azure AKS is secondary "for
  disaster recovery … If AWS fails or becomes degraded, traffic can be recovered
  by serving it from Azure." Route 53 can run active-passive or active-active by
  latency, and "since cost is not a constraint, the documented approach is
  active-active so that performance between cloud providers can also be
  compared."
- Constitution principle 12: the full profile "MUST use AWS EKS as primary and
  Azure AKS as a synchronized active-active target routed by Route 53."

So the original rationale had two parts: survive an AWS failure, and split real
traffic between both clouds by latency to compare them. The second part is the
"distribute Internet traffic between clouds" the maintainer remembered.

## Evaluation against 2026-09-11

**1. The premise of active-active is false today.** It rests on "cost is not a
constraint." Cost is now the binding constraint: the AWS side at full
simultaneity is about USD 1.28 an hour, the Azure subscription is a
credit-limited student subscription with six vCPUs per region, and the
maintainer is scheduling around budget.

**2. Active-active is functionally wrong for this application.** Todos live in
process memory, users in pod-local H2, and Redis is neither persisted nor
replicated. Latency routing sends a user's successive requests to whichever cloud
answers faster, and each cloud has its own data. A user would see todos appear
and disappear, and an account created on one cloud would not exist on the other.
Sharing the JWT secret, which spec 009 T133 checks, makes tokens portable; it
does not make data consistent.

**3. Comparing providers does not need production traffic.** A performance
comparison is a benchmark: repeatable synthetic load against both clusters,
on demand. Splitting real users to obtain it costs correctness (point 2) and
money (point 1).

**4. The failure that has actually happened is the loss of an AWS account.**
The project has run in three AWS accounts in three weeks. Each loss took the
economical platform, its state, its images, and its secrets with it. A second
AWS region does not help against that; a second provider, with its own billing
and identity, does. This is the strongest technical reason for multicloud the
project has, and it was not in the original rationale.

**5. A region failure is the other real risk** — the maintainer's concern that
one `us-east-1` outage takes everything down. As designed, it does.

**6. Complexity is real and must buy something.** Two clouds double the IaC,
pipelines, identity systems, networks, secrets, observability, and the ways
environments can diverge. It is justified only by risks a single cloud cannot
cover — points 4 and 5 — not by points 1 to 3.

## Alternatives

| Option | Survives region loss | Survives AWS account loss | Complexity | Fits budget |
| --- | --- | --- | --- | --- |
| A. One AWS region | No | No | Lowest | Yes |
| B. Two AWS regions | Yes | No | Medium; `us-east-2` allows only 5 On-Demand and 5 Spot vCPUs in this account | Tight |
| C. AWS primary, Azure recovery domain, active-passive | Yes | Yes | Medium | Yes |
| D. AWS and Azure active-active by latency (original) | Yes | Yes | Highest | No; and incorrect without replicated data |

## Decision (proposed)

**Option C: AWS is the single primary platform; Azure is an independent
recovery domain.** Multicloud exists to survive the loss of the AWS region or
the AWS account — not to spread load, and not to compare providers with real
users.

### What lives where

| Capability | AWS (`us-east-1`) | Azure (Central US or South Central US) |
| --- | --- | --- |
| Economical platform (`eco`) | Yes | No |
| Full profile `fdev`, `fstg`, `fprd` | Yes | No |
| Production recovery | — | Warm-standby AKS for `fprd`, sized within six vCPUs, independently reconciled by its own ArgoCD from GitHub |
| Image publication | ECR | ACR mirror of every production digest, copied without rebuilding |
| Secrets | Secrets Manager | Key Vault with the four production secrets |
| Off-provider backups | — | Blob containers holding Terraform state replicas and Velero backups of `eco` |
| Public DNS | See below | See below |
| Performance comparison | Synthetic load on demand | Synthetic load on demand |

### Mode

- **Active-passive.** All production traffic goes to AWS; health-checked failover
  moves it to Azure only when AWS fails.
- **Active-active by latency** stays out until a replicated data store makes a
  user's requests consistent across clouds. That prerequisite is named in the
  constitution rather than left implicit.

### DNS cannot live only in the account it must survive

The public zone is in Route 53 inside the AWS account. If the account is lost,
the zone goes with it and nothing can fail over. Two ways out:

1. Serve the zone from both Route 53 and Azure DNS, with the registrar delegating
   to both sets of name servers, and keep records identical through Terraform.
2. Keep Route 53 and document a manual switch at the registrar, accepting a
   recovery time measured in hours.

Option 1 is recommended once the Azure estate exists; option 2 is acceptable
until then, and must be written into the recovery plan either way.

### Region placement in AWS

With the Azure recovery domain covering production, `fprd` does not need a second
AWS region. After deleting the unused default VPC, `us-east-1` holds `eco`,
`fdev`, `fstg`, `fprd`, and the egress hub within every quota (see
`full-platform/capacity-and-regions.md` §3, corrected on this date). The
`us-east-2` proposal in that document is withdrawn: this account allows only five
vCPUs there.

### Recovery objectives

- **RPO**: the last backup for `eco`; zero for images and secrets already
  mirrored; in-memory application data is lost on failover, and that loss is
  disclosed, as constitution principle 12 already requires.
- **RTO**: the time to scale the warm-standby node pool and switch DNS — minutes
  with dual-provider DNS, hours with a manual registrar change.

## Consequences

- Constitution principle 12 changes from "synchronized active-active" to
  "independent recovery domain, active-passive", with active-active gated on
  replicated data. The accompanying amendment carries the text.
- Spec 009 user story 5 — acceptance scenario 1 and task T140 — changes from
  latency routing to failover routing. That is part of the spec 009 amendment
  already pending as decision 1.
- The economical profile gains off-provider backups at almost no compute cost.
- The full profile no longer needs a second AWS region.
- The Azure estate stays small enough for the student subscription.
