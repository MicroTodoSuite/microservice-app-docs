# Sequence diagrams

**Status**: current as of 2026-09-21.
**Companion to**: [Architecture diagrams](Architecture%20diagrams.md), which shows the
structure. This page shows the flows that structure carries.

Every diagram below describes what is implemented today. Where a step exists only
in the full profile, or only as a plan, it says so.

The same six diagrams are drawn in draw.io, one page each, in
[MicroTodoSuite sequence diagrams.drawio](assets/MicroTodoSuite%20sequence%20diagrams.drawio),
rendered from this page. A change to a sequence here needs the same change there.

## A commit becomes a signed image

The reusable workflow in `MicroTodoSuite/.github` builds an image once. Nothing
downstream ever rebuilds it; every later stage refers to the digest this run
produced.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Service maintainer
    participant GH as GitHub service repo
    participant CI as Reusable ci.yml
    participant AWS as AWS STS OIDC
    participant ECR as Amazon ECR
    participant REL as semantic-release

    Dev->>GH: push branch, open pull request
    GH->>CI: run tests, SonarCloud, Trivy
    CI-->>GH: required gates pass
    Dev->>GH: merge to main
    GH->>CI: run on main
    CI->>CI: build image once
    CI->>AWS: assume lex-mts-shd-role-ecrpublish (no static keys)
    AWS-->>CI: short-lived credentials
    CI->>ECR: push image, receive digest
    CI->>CI: Syft SBOM, Cosign keyless signature
    CI->>ECR: attach SBOM and signature to the digest
    CI->>REL: semantic-release tags the version
    REL-->>GH: release published
```

## A digest reaches production

Promotion never rebuilds and never moves a tag. One pull request per environment
changes one overlay's digest, and production waits for a human.

```mermaid
sequenceDiagram
    autonumber
    participant REL as Release in service repo
    participant PROM as Reusable promote.yml
    participant ECR as Amazon ECR
    participant GO as microservice-app-gitops
    participant REV as Reviewer
    participant ARGO as ArgoCD in cluster

    REL->>PROM: digest, profile, environment, destination
    PROM->>ECR: cosign verify against the Actions OIDC issuer
    ECR-->>PROM: signature valid
    PROM->>GO: bump-image.sh rewrites only images[].digest
    PROM->>GO: open promotion pull request (dev)
    REV->>GO: review and merge
    GO->>ARGO: dev overlay now names the new digest
    Note over PROM,GO: staging repeats the same two steps
    PROM->>GO: open promotion pull request (prod)
    REV->>REV: gate-prod environment approval
    REV->>GO: merge
    GO->>ARGO: prod overlay now names the new digest
```

## The production canary decides for itself

Production is an Argo Rollouts canary. The gate is real traffic measured in
Prometheus, not a synthetic probe.

```mermaid
sequenceDiagram
    autonumber
    participant GO as Git prod overlay
    participant ARGO as ArgoCD
    participant RO as Argo Rollouts
    participant SVC as canary Service
    participant PROM as Prometheus
    participant AN as AnalysisRun

    GO->>ARGO: new digest committed
    ARGO->>RO: Rollout spec updated
    RO->>SVC: shift 10% of traffic to the canary revision
    RO->>AN: start microtodosuite-canary-health
    AN->>PROM: error ratio for this workload, revision="canary"
    PROM-->>AN: value (NaN when no traffic yet)
    alt ratio <= 5% or no traffic
        AN-->>RO: Successful
        RO->>SVC: 25%, then 50%, then 100%
        RO->>RO: canary becomes stable
    else ratio > 5% on five consecutive measurements
        AN-->>RO: Failed
        RO->>SVC: shift all traffic back to stable
        RO->>RO: Rollout aborted, stable revision serving
    end
```

## Rollback is a revert

There is no rollback button and no `kubectl` step. The cluster follows Git, so
undoing a release is undoing a commit.

```mermaid
sequenceDiagram
    autonumber
    actor Op as Operator
    participant GO as microservice-app-gitops
    participant ARGO as ArgoCD
    participant K8s as Cluster

    Op->>GO: git revert of the promotion commit
    GO->>ARGO: main now names the previous digest
    ARGO->>ARGO: detect OutOfSync
    ARGO->>K8s: apply the previous digest
    K8s-->>ARGO: Healthy on the previous revision
    Note over Op,K8s: A direct kubectl apply is forbidden; it would be reverted by the next sync
```

## A profile goes down and comes back

`scripts/aws-profile-lifecycle.sh` in `microservice-app-ops` is the only way the
runtime is created or destroyed. Applies use a saved plan and nothing else, and
the way down needs no GitOps pull request.

```mermaid
sequenceDiagram
    autonumber
    actor Op as Operator
    participant LC as aws-profile-lifecycle.sh
    participant EC2 as EC2 API
    participant TF as Terraform
    participant AWS as AWS
    participant GO as microservice-app-gitops

    Note over Op,GO: Down
    Op->>LC: snapshot-volumes
    LC->>EC2: snapshot every EBS CSI volume
    Op->>LC: quiescence-receipt with the volume record
    LC->>EC2: dry-run inventory of the sweepable runtime resources
    LC-->>Op: checksummed receipt, nothing mutated
    Op->>LC: plan down with the receipt and the volume record
    LC->>LC: refuse a tampered, mismatched or out-of-order receipt
    LC->>TF: saved plans, JSON evidence and checksums
    Op->>LC: apply down, exact saved plans only
    LC->>AWS: destroy the cluster, the workload bundle
    LC->>EC2: post-destroy sweep of what the controllers left behind
    Note over LC,EC2: listeners, load balancers, target groups, orphaned ENIs, security groups, then snapshotted volumes, each revalidated just before deletion
    LC->>AWS: destroy the runtime network, keep the durable layer
    Note over Op,GO: Up
    Op->>LC: plan up, then apply up
    LC->>AWS: recreate VPC, cluster and nodes
    Op->>GO: bootstrap-cluster.sh, the two audited mutations
    GO->>AWS: ArgoCD reconciles the unchanged desired state
```

> Until microservice-app-ops#122 (2026-09-20) the way down needed a merged GitOps
> quiescence pull request, named by `--gitops-revision`. The receipt replaces it,
> and the sweep removes the load balancers, security groups, network interfaces
> and volumes the cluster's controllers created, which Terraform does not own.

## A secret reaches a pod without touching Git

No secret value is ever committed. The cluster reads AWS Secrets Manager with an
identity bound to the pod's ServiceAccount.

```mermaid
sequenceDiagram
    autonumber
    participant TF as Terraform
    participant SM as AWS Secrets Manager
    participant ESO as External Secrets Operator
    participant STS as AWS STS
    participant K8s as Kubernetes Secret
    participant Pod as Workload pod

    TF->>SM: create the secret and the IRSA role
    Note over TF,SM: Git holds the name and the role ARN, never the value
    ESO->>STS: assume the IRSA role with its ServiceAccount token
    STS-->>ESO: short-lived credentials
    ESO->>SM: read the secret value
    SM-->>ESO: value
    ESO->>K8s: create or refresh the Secret
    Pod->>K8s: mount it as an environment variable
```
