# Architecture Review: Eraser Targets Versus Validated Workspace State

**Review date:** 2026-08-09
**Scope:** MicroTodoSuite target architecture, local GitOps pilot, local platform
add-ons, and the Terraform foundation currently present in `microservice-app-ops`

## 1. Sources and classification rules

The Eraser sources reviewed through the Eraser MCP were:

- [Full / expensive architecture](https://app.eraser.io/workspace/yiQVIvN8Z18hOsC6ea5N),
  diagram `OV-66hw4OPmr3GD6OQt-O`, titled *Arquitectura Multicloud de
  Microservicios con Kubernetes*.
- [Cost-optimized / economical architecture](https://app.eraser.io/workspace/pzSJKHfGsYeR4VeFP0fO),
  diagram `1D02-dnb9M4ZpHe1syH4`, titled *Arquitectura Cloud Optimizada con EKS
  Único*.

The repository evidence reviewed was:

- `microservice-app-docs/constitution.md` and
  `docs/MicroTodoSuite evolution plan.md`;
- `microservice-app-gitops/docs/reconciliation-notes.md`,
  `docs/platform-addons.md`, `docs/profiles.md`, and the local pilot documents;
- every current `specs/*/checklists/` file;
- the successful pilot evidence under
  `.local/evidence/runs/20260809T045303Z-initial-deploy/`;
- the add-on evidence under
  `.local/evidence/platform-addons/20260809T165041Z/`;
- the retained failed/overrun evidence directories, so they were not mistaken
  for successful acceptance records; and
- the Terraform roots, workflows, Git refs, and history in
  `microservice-app-ops`.

The requested `microservice-app-gitops/docs/reconciliation-notes-v2.md` does
not exist in the tracked tree, local evidence directories, or Git history. The
platform add-on acceptance record explicitly states that it did not use that
file as evidence. This review therefore treats `docs/reconciliation-notes.md`
as a historical pre-remediation assessment and uses the later raw pilot and
platform-add-on evidence as the authoritative record of what subsequently
passed. The missing v2 report remains an evidence-documentation gap.

Each diagram element is assigned one of these classifications:

- **ALREADY MATCHES WHAT IS BUILT**: the element or contract has concrete
  implementation evidence; the note states whether that evidence is local-live,
  static, application-code-only, or legacy Azure evidence.
- **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT**: the element is absent or
  only scaffolded, but the validated contracts do not prevent it.
- **INCONSISTENT WITH WHAT IS ALREADY VALIDATED**: the diagram states a behavior
  that conflicts with a proven runtime or immutable contract.

Repeated service instances and equivalent connectors in dev, staging, prod,
and DR are grouped by logical element below. The grouping covers every node and
relationship in both diagrams without pretending that a repeated drawing is a
separately validated deployment.

## 2. What has actually been built and validated

| Capability | Evidence-backed state | Limit of the evidence |
| --- | --- | --- |
| Local GitOps mechanism | **Validated narrowly.** A machine-local Git source, loopback registry, kind cluster, ArgoCD, environment Application, and one `auth-api` Application reconciled automatically. Revision `a6f9c241...` became Synced/Healthy in 44 seconds; one digest-pinned pod became Ready; three `/version` calls returned HTTP 200 over 60 seconds. | This is one successful initial-deploy record, not closure of the complete `specs/001-local-gitops-pilot` acceptance program. That spec has no acceptance checklist in the current tree, and its task list remains unchecked. |
| GitOps-only change path | **Validated after the documented bootstrap boundary.** The only direct applies in the pilot evidence installed ArgoCD and its root Application. Managed workload and add-on changes were commits reconciled by ArgoCD. | Bootstrap is a narrow exception, not a general deployment path. |
| Immutable workload contract | **Validated live.** `auth-api` ran as `localhost:5001/auth-api@sha256:...`; Kyverno later reported `pass` for both the digest and health-probe policies. | Cosign signature verification and cross-registry promotion were not tested. |
| KEDA | **Validated live**, version 2.20.1. Its expected Deployments were Available and the capability ScaledObject was Ready/Active with an HPA. | No MicroTodoSuite Redis-queue or Prometheus-driven business scaling was validated. |
| cert-manager | **Validated live**, version 1.21.0. Its three Deployments were Available and a self-signed Certificate produced a Ready TLS Secret. | No public issuer, DNS challenge, Route 53 integration, or production ingress certificate was validated. |
| External Secrets Operator | **Validated live**, version 2.9.0. Its three Deployments were Available and `ExternalSecret/auth-api-secrets` was Ready with `SecretSynced`. | The shared installation is provider-neutral. AWS Secrets Manager, Azure Key Vault, IRSA, and AKS workload identity were not validated. |
| Kyverno | **Validated live**, version 1.18.2. Four Deployments were Available; digest and liveness/readiness policies were Enforce/Ready and passed for the fresh `auth-api` pod. | Cosign signature admission and a broader production policy set were not validated. |
| Dual-profile plumbing | **Built and statically exercised.** `topology-economical`, `topology-full`, cluster registration seams, replica-based Rollout manifests, and a Prometheus AnalysisTemplate exist for `auth-api`. | Istio and Argo Rollouts controllers were not installed; the canary component remains inactive. A sidecar-injection annotation is a contract, not a service mesh. |
| Application estate | The five application repositories and their languages match the diagrams. Frontend-to-auth/todos, auth-to-users, and todos-to-Redis-to-log-processor behavior exists in source. | Only `auth-api` was part of the retained successful local-live evidence. Other service onboarding work appeared as concurrent, uncommitted work during this review and is not credited as validated. |
| AWS Terraform foundation | **Not built.** No AWS provider, AWS resource, EKS, VPC, ECR, S3 backend, DynamoDB lock table, IAM/IRSA, Route 53, Karpenter, or AWS Secrets Manager Terraform appears in the current branch or searched Terraform history. | `microservice-app-ops` still provisions Azure Blob state, ACR, Log Analytics, one Azure Container Apps environment, and ten Container Apps. |
| AKS foundation | **Not built.** There is no AKS module or separate AKS backend. | Existing Azure Terraform is for Container Apps, not AKS. |

The four-add-on record is a strong PASS at local revision `111186a8...`: the
four infrastructure Applications and `auth-api-local` were Synced/Healthy at
the same revision, all 14 expected add-on Deployments were Available, all final
pods were Running/Ready, and three HTTP checks passed over 61 seconds. It does
not validate any fifth add-on, cloud binding, managed cluster, or full-profile
networking component.

## 3. Elements common to both diagrams

| Diagram element or relationship | Classification | Comparison with implementation and evidence |
| --- | --- | --- |
| Browser users and the Vue frontend | **ALREADY MATCHES WHAT IS BUILT** | The frontend repository and browser application exist. Its code calls same-origin `/login` and `/todos`. The target Kubernetes exposure is not live; the local contract permits only operator port-forwarding. |
| `auth-api` (Go/JWT), `todos-api` (Node), `users-api` (Java), frontend (Vue), and log-message-processor (Python) | **ALREADY MATCHES WHAT IS BUILT** | The service identities and technologies match the codebase. Only `auth-api` has retained local-live Kubernetes evidence. Repetition across three namespaces or two clouds is not yet built. |
| Frontend -> auth-api and frontend -> todos-api | **ALREADY MATCHES WHAT IS BUILT** | The frontend implements `/login` and `/todos` calls. The full end-to-end local proxy path is specified for feature 004 but has not yet produced acceptance evidence. |
| auth-api -> users-api | **ALREADY MATCHES WHAT IS BUILT** | `auth-api` reads `USERS_API_ADDRESS` and requests `/users/{username}` before issuing a JWT. The isolated pilot used only `/version`, so this relationship is code-backed but not yet local-live validated. |
| todos-api -> Redis -> log-message-processor | **ALREADY MATCHES WHAT IS BUILT** | todos-api publishes operation JSON to the configured Redis channel and the processor subscribes to that channel. Redis and the two consumers were not in the retained passed pilot run. |
| todos-api -> users-api | **INCONSISTENT WITH WHAT IS ALREADY VALIDATED** | Both Eraser diagrams draw this HTTP edge, but the todos implementation validates JWT locally and has no users-api call. The current onboarding contract lists Redis as todos-api's required service and no users-api dependency. This connector must not be claimed as current behavior. |
| `microservice-app-gitops` as desired-state source | **ALREADY MATCHES WHAT IS BUILT** | The local pilot and four add-ons used commits followed by automatic ArgoCD reconciliation. |
| ArgoCD watch/sync, prune, and self-heal | **ALREADY MATCHES WHAT IS BUILT** | Proven for the local root, environment, `auth-api`, and four add-on Applications. Cross-cluster control from EKS prod is not built. |
| GitHub Actions build/test/scan/sign/auto-PR pipeline | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Existing service workflows are legacy, duplicated, and deploy directly to Azure Container Apps. The pilot proves the desired digest-commit-reconcile contract, but no reusable CI workflow, complete test matrix, Trivy/Syft/Cosign chain, or GitOps PR automation has replaced the legacy path. |
| `microservice-app-ops` as Terraform owner | **ALREADY MATCHES WHAT IS BUILT** | Terraform ownership exists, but only for the legacy Azure Container Apps platform. The AWS, EKS, and AKS resources shown in Eraser are absent. |
| KEDA, cert-manager, External Secrets Operator, and Kyverno | **ALREADY MATCHES WHAT IS BUILT** | These are the exact four add-ons installed and functionally validated on kind through the shared provider-neutral ApplicationSet. Their production bindings remain deferred. |
| Falco, Chaos Mesh, and OpenCost | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No installation folder, live Application, controller evidence, or capability evidence exists for any of them. The shared add-on registration pattern can accommodate them. |
| OpenTelemetry Collector and SDK in every service | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Current services use Zipkin instrumentation and Prometheus metrics. No OTel collector or complete SDK migration is installed or validated. |
| Prometheus and Grafana in the target Kubernetes platform | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Both components exist in the legacy Azure Terraform/application estate, and several services expose Prometheus metrics. They were not deployed or tested by the local GitOps evidence, so legacy component existence is not treated as a target-platform deployment. |
| Jaeger | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Current tracing is Zipkin. Neither full Jaeger nor embedded-retention Jaeger is present in the GitOps pilot. |
| Alertmanager -> Slack | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Alertmanager deployment, Slack integration, or notification evidence exists. |
| ESO connection to a cloud secret store | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The provider-neutral ESO controller and stable Secret target contract are proven. AWS Secrets Manager and Azure Key Vault bindings are explicitly destination work and remain untouched. |

The inaccurate todos-api -> users-api connector does not prove that either
overall architecture is infeasible. The lowest-risk action is to let feature
004's functional acceptance confirm the real call graph, then remove the
connector from both diagrams unless a later approved specification deliberately
adds that dependency.

## 4. Full / expensive diagram

| Full-diagram element | Classification | Comparison with implementation and evidence |
| --- | --- | --- |
| One AWS account | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | It is a binding suite convention, but no AWS account integration is represented in Terraform or live evidence. |
| VPC dev, staging, and prod | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No AWS VPC module or resource exists. The current cluster-registration contract is designed to accept future endpoints without changing service bases. |
| EKS dev, staging, and prod | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No EKS cluster exists. Managed overlays and full/economical topology components are inactive scaffolds only. |
| Amazon ECR | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The local registry proves digest-based publication, but no ECR repository, authentication, or promotion path exists. |
| S3 + DynamoDB Terraform state and locking | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | `microservice-app-ops` uses Azure Blob remote state. No AWS backend foundation exists. |
| AWS Secrets Manager | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | ESO's provider-neutral seam is validated; the AWS store and IRSA binding are not. |
| Karpenter node group | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No EKS nodes, Karpenter controller, NodePool, or EC2 evidence exists. |
| AWS Application Load Balancer | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The local pilot deliberately uses port-forwarding and no cloud ingress. |
| Amazon Route 53 latency routing and health checks | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Route 53 Terraform resource, hosted zone, record, health check, or live DNS test exists. Route 53 fields found inside the vendored cert-manager CRDs are schema text, not configured Route 53. |
| Azure Resource Group and AKS `aks-dr` | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The existing Azure resource group is part of the legacy Container Apps platform. There is no AKS module, cluster, registration, or DR deployment. |
| Azure Load Balancer | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No AKS load balancer or ingress path exists. |
| Azure Container Registry mirror | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | ACR exists for the legacy platform, but no ECR-to-ACR promotion or AKS consumption path is implemented; the target mirror is absent. |
| CI publishes the image to both ECR and ACR | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No dual-registry pipeline exists, but the immutable-image and registry-seam contracts permit it. |
| The ECR/ACR “same tag” label as the promotion guarantee | **INCONSISTENT WITH WHAT IS ALREADY VALIDATED** | The live pilot and Kyverno evidence require `sha256` deployment references, and the onboarding contract requires identical manifest/index digests across registries. The same tag may remain as a human-readable alias, but it cannot replace digest equality as the guarantee. |
| Azure Key Vault and Azure Storage backend for AKS | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Azure Storage exists only for legacy Terraform state. No AKS backend or Key Vault/ESO binding is present. |
| ArgoCD control plane in EKS prod, remotely syncing other EKS clusters and AKS | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Local in-cluster reconciliation is proven. Remote cluster registration, credentials, reachability, and failure behavior are untouched. |
| Istio ingress gateways, service meshes, Envoy sidecars, and mTLS in EKS prod and AKS | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Istio is present in the full diagram. No Istio controller, CRD, Gateway, VirtualService, DestinationRule, certificate, Kiali integration, or live sidecar exists. `sidecar.istio.io/inject: "true"` in `topology-full` is only prepared workload metadata. |
| Kiali in EKS and AKS | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Kiali installation or mesh telemetry exists. |
| Argo Rollouts canary in EKS prod | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | A Rollout and Prometheus AnalysisTemplate are statically scaffolded for auth-api, but the controller is an inactive placeholder and no canary has run. Exact traffic percentages additionally depend on the unbuilt Istio layer. |
| Argo Rollouts canary in AKS | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | There is no AKS runtime evidence. However, the diagram conflicts with the current evolution-plan text, which says `aks-dr` receives the prod-validated release through a rolling update rather than repeating the canary. Resolve this source-of-truth mismatch before implementation; it is not a runtime-proven reason to redesign multicloud. |
| Complete add-on set duplicated across AWS and Azure | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Reuse of the same four provider-neutral roots is supported by the validated registration contract. Istio/Kiali, Falco, Chaos Mesh, OpenCost, and destination-specific bindings remain absent. |
| Full ELK/Filebeat pipeline in each cloud | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Filebeat, Logstash, Elasticsearch, or Kibana manifest or live evidence exists. |
| Full-cloud OTel -> Jaeger/Prometheus/ELK and Alertmanager paths | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No target observability stack was part of the pilot or add-on acceptance run. |

### Full-profile conclusion

The full diagram remains a feasible target and remains the constitution's
default profile. Nothing in the local evidence proves that Istio, AKS, Route 53,
or active-active multicloud must be removed. They are simply unimplemented and
must not be described as delivered. No topology change is justified. The only
directly evidence-mandated notation correction is to make the same immutable
OCI manifest/index digest—not merely the same tag—the cross-registry promotion
guarantee. Signature verification remains planned rather than locally proven.

## 5. Cost-optimized / economical diagram

| Economical-diagram element | Classification | Comparison with implementation and evidence |
| --- | --- | --- |
| One AWS account, one multi-AZ VPC, one EKS cluster | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The local pilot validates one-cluster mechanics, not AWS infrastructure, multiple AZs, or EKS. Running this managed profile still requires the constitutionally required profile approval. |
| dev, staging, and prod as namespaces | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The local pilot uses only `microtodo-local`. Namespace activation and value-only registration are prepared, but no managed three-namespace EKS deployment has been validated. |
| ResourceQuota, LimitRange, NetworkPolicy, and RBAC per managed namespace | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Local quota, limit, and network-policy patterns exist, and the pilot uses token-disabled service accounts and constrained AppProjects. Complete per-namespace RBAC and all three managed environments are not validated, so the target element is not classified as already built. |
| Amazon Route 53 simple DNS -> one ALB | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Route 53 or ALB resource exists. This is distinct from the full diagram's active-active latency routing and can be deferred until a real EKS ingress endpoint exists. |
| Public ALB -> NGINX Ingress Controller | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The local service contract intentionally uses port-forwarding and forbids adding a local-only ingress controller. A managed-environment NGINX/ALB binding remains an environment-owned future capability. |
| No Istio/service mesh | **ALREADY MATCHES WHAT IS BUILT** | The local pilot is explicitly economical and has no mesh. This validates only the no-mesh choice locally; it does not validate the managed EKS architecture. |
| No AKS or multicloud DR | **ALREADY MATCHES WHAT IS BUILT** | Local work has not touched AKS or multicloud. This absence is intentional in the economical profile, not evidence against the full profile. |
| Five services repeated in each namespace | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | The repositories exist and the reusable shape is prepared. Only auth-api has passed retained local-live evidence; current service-onboarding drafts are not an acceptance result. |
| In-code circuit breaker, retry, bulkhead, and timeout | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Redis retry behavior exists in todos-api, but there is no suite-wide resilience-library implementation or evidence for the four named patterns. |
| Native replica-based Argo Rollouts canary in prod | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Static Rollout/AnalysisTemplate plumbing exists, but the controller is not installed and prod is inactive. This is the appropriate canary mode to validate before adding Istio traffic routing. |
| ECR, S3/DynamoDB state, and AWS Secrets Manager | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | None exists in Terraform. Local registry, Git state, and provider-neutral ESO prove interfaces only. |
| Karpenter with a high Spot proportion | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Karpenter or EC2 capacity exists. |
| Velero backups to S3 / cold DR | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Velero installation, backup location, backup, restore, or S3 target exists. |
| KEDA, cert-manager, ESO, and Kyverno | **ALREADY MATCHES WHAT IS BUILT** | These four controllers and their local capability checks are validated. Production stores, issuers, ingress, and workload scaling remain deferred. |
| Falco, Chaos Mesh, and OpenCost | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | These were not among the four validated add-ons and have no manifests or evidence. |
| One shared OTel/Prometheus/Grafana/Jaeger/Loki/Alertmanager stack | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | Shared placement is compatible with one cluster, but none of this target stack was validated locally. Legacy Prometheus/Grafana and Zipkin do not prove this pipeline. |
| Promtail -> Loki | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Promtail or Loki implementation exists. |
| Jaeger embedded storage | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No Jaeger implementation exists; current tracing uses Zipkin. |
| Alertmanager -> Slack | **NOT YET BUILT BUT CONSISTENT WITH WHAT IS BUILT** | No notification path is installed or tested. |

### Economical-profile conclusion

The economical diagram is topologically closer to the local kind pilot because
both use one cluster and no service mesh. That does not mean the economical AWS
architecture has been built, and it does not silently select that profile for a
managed environment. The constitution and `docs/profiles.md` require an approved
architecture amendment before a managed environment actually runs in economical
mode.

## 6. Explicitly untouched elements

The local work has **not** validated any of the following:

1. **Istio or Kiali.** There is no installed mesh, ingress gateway, mTLS,
   traffic-policy resource, or sidecar observation. Full-topology annotations
   and comments are scaffolding only.
2. **AWS EKS foundations.** There is no VPC, EKS, ECR, IAM/IRSA, S3 backend,
   DynamoDB lock table, ALB, Karpenter, or AWS Secrets Manager implementation.
3. **Azure AKS or multicloud.** There is no AKS module, AKS cluster, ACR mirror,
   Key Vault binding, cross-cluster ArgoCD registration, or DR evidence.
4. **Route 53.** Neither simple DNS nor latency/health-check routing exists.
   Vendor CRD documentation that mentions Route 53 is not a configured resource.
5. **Argo Rollouts runtime.** The controller is absent; no replica-based or
   Istio-percentage canary has executed.
6. **Managed ingress and TLS.** There is no ALB, NGINX Ingress Controller, Istio
   Gateway, public issuer, DNS challenge, or externally verified certificate.
7. **Target observability.** OTel, Jaeger, ELK/Filebeat, Loki/Promtail,
   Alertmanager, and Slack notification paths are absent from the validated
   local platform.
8. **Remaining platform services.** Falco, Chaos Mesh, OpenCost, Velero, and the
   production uses of KEDA/cert-manager/ESO/Kyverno remain unvalidated.
9. **Cloud supply chain.** Reusable CI, OIDC, ECR/ACR promotion, Trivy, Syft,
   Cosign, and signature admission have not run end to end.

Consequently, “four platform add-ons validated” must not be summarized as
“roadmap platform task complete”: Istio is one of the explicitly named initial
add-ons and is still untouched, while Falco, Chaos Mesh, and OpenCost belong to
the broader target platform and are also absent.

## 7. Recommended execution sequence

This sequence adjusts delivery order and evidence gates while preserving both
target architectures.

1. **Finish and evidence the current local service-onboarding feature.** Follow
   its existing dependency order: reconcile and prove Redis first; then validate
   users-api with auth-api; then todos-api plus log-message-processor; then the
   frontend same-origin path. Do not credit uncommitted manifests as built until
   the exact-revision, immutable-image, Redis `PONG`, login, todo-event, frontend,
   and regression evidence passes.
2. **Close the local pilot's evidence-documentation debt.** Produce the missing
   post-remediation reconciliation record and reconcile the current pilot task
   and acceptance state with the raw successful run. Keep historical failed
   runs, but distinguish them from final evidence.
3. **Validate Argo Rollouts locally in economical mode.** Install the pinned
   controller through GitOps, activate a replica-based canary for one service,
   and prove metric-gated progression and Git-revert recovery. This exercises
   the delivery contract without pretending Istio exists.
4. **Add the provider-neutral economical observability slice locally.** Start
   with OTel, Prometheus/Grafana, short-retention Jaeger, and Loki/Promtail, then
   prove service telemetry and alerts. Defer ELK and multi-cloud duplication.
5. **Specify and build the AWS Terraform foundation in small cloud increments.**
   `microservice-app-ops` needs a full Spec Kit lifecycle before implementation.
   Preserve the legacy Azure roots, then build the AWS state and shared modules,
   followed by one EKS dev/VPC/ECR/IAM foundation as the first integration unit;
   do not attempt three EKS clusters and AKS in one change.
6. **Register EKS dev through the already validated GitOps seam.** Bind ECR,
   Secrets Manager/IRSA, a managed issuer, and ingress as destination-owned
   values. Re-run the same add-on and application evidence at the cloud revision.
7. **Replace legacy CI deployment with immutable GitOps promotion.** Build once,
   test and scan, create the SBOM, sign the image, push/copy the same OCI
   manifest/index, and open a GitOps PR that selects the digest. Remove direct
   application mutation from CI only after the replacement path is proven.
8. **Implement the selected managed profile without changing the diagrams by
   default.** Without an approved economical-profile amendment, continue the
   full default: replicate the proven Terraform/GitOps integration to separate
   staging and prod EKS/VPCs. If the economical profile is approved instead,
   keep the single EKS cluster and prove all namespace isolation controls.
9. **Introduce Istio only on the full-profile EKS dev integration.** Install
   Istio and Kiali through the add-on mechanism, then validate gateways, sidecar
   injection, mTLS, and resilience. Only after that should Argo Rollouts add
   exact traffic percentages and the full profile advance to production.
10. **Defer AKS, ACR mirroring, and Route 53 active-active routing until AWS prod
    is stable.** Build the separate AKS backend/module, deploy the production-
    validated digest with the approved DR strategy, disclose state loss, and
    prove health-based failover before enabling latency routing. Add full-cloud
    chaos exercises only after the basic DR path works.
11. **Add the remaining platform controls behind their own evidence gates.**
    Falco, Chaos Mesh, OpenCost, Velero, and the full ELK path should each have a
    pinned installation, provider boundary, capability check, reconciliation
    record, and rollback proof rather than being enabled as one undifferentiated
    bundle.

## 8. Architecture decision

No evidence supports removing Istio, AKS, Route 53, multicloud, or either cost
profile from the architecture. They are deferred targets, not failed designs,
and no architecture-topology change is recommended.

One narrow notation correction is required now because it is directly
contradicted by the live immutable-image policy and the repository contract:
the full diagram must state that ECR and ACR promotion preserves the same OCI
manifest/index digest. A shared tag may be retained only as an alias. The
todos-api -> users-api connector should be treated as an accuracy defect
pending feature 004's live confirmation, not as a reason to redesign the
service estate.

The principal correction is therefore to the execution plan and claims of
completion: finish locally testable behavior first, preserve immutable GitOps
contracts, introduce one cloud integration at a time, and defer full-profile and
multicloud components until their prerequisites have passed.
