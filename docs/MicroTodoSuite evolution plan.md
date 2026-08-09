# Consolidated Migration Plan for MicroTodoSuite: From Azure Container Apps to AWS EKS

With Azure AKS as the secondary multicloud platform, ArgoCD as the continuous delivery layer, and Spec-Driven Development as the team's working methodology.

## 1. Guiding Principles

1. A single AWS account. Environment isolation is achieved through separate clusters and VPCs, not separate accounts.
2. GitOps as the only deployment path. Every change goes through a commit to the `gitops` repository, which ArgoCD reconciles.
3. Trunk-Based Development across all repositories, with short-lived branches and feature flags for incomplete functionality.
4. The specification is the source of truth for the work performed by both the team and AI agents, rather than standalone prompts or isolated conversations.
5. Cost does not constrain the design, but it is measured through FinOps so that simplifying something remains an informed decision.

## 2. Repository Structure

| Repository                               | Content                                                |
| ---------------------------------------- | ------------------------------------------------------ |
| `microservice-app-auth-api`              | Go service + JWT                                       |
| `microservice-app-todos-api`             | Node.js service                                        |
| `microservice-app-users-api`             | Java/Spring Boot service                               |
| `microservice-app-frontend`              | Vue.js                                                 |
| `microservice-app-log-message-processor` | Python, Redis consumer                                 |
| `microservice-app-ops`                   | Terraform: VPCs, EKS clusters, IAM/IRSA, ECR, Route 53 |
| `microservice-app-gitops`                | New. ArgoCD source of truth                            |
| `microservice-app-ai-agents`             | New. Project agents, skills, and MCPs                  |
| `microservice-app-docs`                  | Documentation, diagrams, SDD constitution              |
| `microservice-app-prometheus`            | Custom Prometheus image                                |
| `.github`                                | Organization profile and shared reusable workflows     |

All repositories use Trunk-Based Development. `ops` maintains the cloud infrastructure and changes infrequently; `gitops` maintains which version of each service runs in each environment and changes daily.

## 3. Infrastructure (Terraform)

A single AWS account with three EKS clusters (`dev`, `staging`, and `prod`), each running in its own VPC. Remote backend in S3 with DynamoDB for locking, and a separate state key per environment within the same bucket. IRSA is used to provide AWS permissions at the pod level. Karpenter is used as the node autoscaler.

Modules:

* `vpc`
* `eks`
* `iam`
* `ecr`
* `route53`

Azure AKS is defined in a separate module with its own backend in Azure Storage. Infracost is integrated into the pipeline to estimate the cost of every Terraform change.

## 4. Kubernetes Platform

| Add-on                    | Purpose                                                                                 |
| ------------------------- | --------------------------------------------------------------------------------------- |
| Istio + Kiali             | Service mesh: mTLS, traffic shifting, canary deployments, circuit breaker/retry/timeout |
| KEDA                      | Event-driven autoscaling using Redis queues and Prometheus metrics                      |
| cert-manager              | Automatic TLS for public ingress                                                        |
| External Secrets Operator | Synchronizes secrets from AWS Secrets Manager / Azure Key Vault                         |
| Kyverno                   | Policy-as-code: blocks unsigned or incorrectly configured images                        |
| Chaos Mesh                | Chaos engineering experiments                                                           |
| Falco                     | Runtime security                                                                        |
| OpenCost                  | Cost per namespace/service, visible in Grafana                                          |

All of these add-ons are also managed through the `gitops` repository.

## 5. GitOps and Continuous Delivery (ArgoCD)

Structure of the `gitops` repository:

```text
microservice-app-gitops/
├── clusters/
│   ├── eks-dev/
│   ├── eks-staging/
│   ├── eks-prod/
│   └── aks-dr/
├── infrastructure/       # Add-ons from section 4, one per folder
├── apps/
│   ├── auth-api/
│   │   ├── base/
│   │   └── overlays/{dev,staging,prod}/
│   ├── todos-api/
│   ├── users-api/
│   ├── frontend/
│   └── log-message-processor/
```

When code is merged into `main` in an application repository, CI builds, tests, and publishes the image, then automatically opens a PR against `gitops` to update the `dev` overlay.

`dev` synchronizes automatically.

Promotion to `staging` and `prod` is performed through another PR that copies the already-tested image tag. The production PR requires manual approval.

Rollback is performed using a `git revert` in the `gitops` repository.

ArgoCD Notifications sends Slack notifications for every synchronization event.

Route 53 distributes traffic based on latency between the AWS cluster and the Azure cluster.

## 6. Deployment Strategy

Recommendation: **progressive canary deployments** in production, managed using Argo Rollouts, which replaces the standard Kubernetes `Deployment`, with different strategies depending on the environment.

* **Dev and staging:** standard Kubernetes rolling updates. Iteration speed is more important than caution because no real users are exposed.
* **Prod:** canary deployment using progressive steps, for example 10%, 25%, 50%, and 100%. Between each step, an Argo Rollouts `AnalysisTemplate` queries Prometheus metrics such as error rate and p99 latency. If those metrics degrade, the deployment automatically rolls back without human intervention. Since the cluster already uses Istio, traffic distribution can be controlled by actual percentages rather than replica ratios.
* **`aks-dr`:** does not repeat the canary process. It receives the version that has already been fully validated and promoted in `eks-prod`, deployed using a simple rolling update so the cost of validating the same release twice is avoided.

Canary deployments are preferred over blue-green deployments because blue-green requires keeping twice the infrastructure running in parallel and traffic switching is all-or-nothing. A problem may therefore only be discovered after 100% of users have been exposed.

Canary deployments expose users gradually, so a problem initially affects only a small fraction of users and automatic rollback can act before the issue scales.

## 7. CI (GitHub Actions)

Reusable workflows are centralized in `.github` and used by the service repositories to eliminate the current duplication.

Authentication to AWS and Azure uses OIDC, with no static credentials.

The container image is built once and promoted between environments.

Mandatory gates for every PR include:

* SonarQube
* Trivy
* Tests described in Section 9

`semantic-release` continues to generate versions and changelogs and now also triggers the PR that updates the `gitops` repository.

Syft generates the SBOM for every image.

Cosign signs the images.

Kyverno verifies the signature before allowing the image into the cluster.

## 8. Design Patterns

Existing patterns are documented, including Retry and External Configuration, and the following patterns are added and implemented at the Istio level:

* Resilience: Circuit Breaker, Bulkhead, Timeout.
* Configuration: Feature Toggle using OpenFeature, and Configuration Server using Spring Cloud Config for `users-api`.

## 9. Testing

| Type                | Tool                                                 |
| ------------------- | ---------------------------------------------------- |
| Unit testing        | `go test`, Jest, JUnit, `pytest`                     |
| Integration testing | Testcontainers                                       |
| Contract testing    | Spectral and Pact against OpenAPI/AsyncAPI contracts |
| E2E                 | Cypress or Playwright against the frontend           |
| Performance         | Locust                                               |
| Security (DAST)     | OWASP ZAP                                            |

Coverage and quality are reported to SonarQube.

## 10. Observability

OpenTelemetry is used as the single instrumentation layer for every service.

Traces are sent to Jaeger.

Technical and business metrics are sent to Prometheus and Grafana.

Logs are sent to ELK:

* Elasticsearch
* Logstash
* Kibana

Filebeat is used as the log collector.

Every Deployment includes:

* Liveness probes
* Readiness probes
* Startup probes

Alertmanager sends notifications to Slack.

## 11. Security

Trivy runs both in CI and continuously inside the cluster.

External Secrets Operator integrates with AWS Secrets Manager.

RBAC is configured per namespace and Service Account, with IRSA providing AWS permissions.

TLS is enabled on every ingress.

Internal mTLS is provided through Istio.

Falco provides runtime threat detection.

`kube-bench` and `kube-hunter` are used for periodic security audits.

## 12. Change Management

Release notes are automatically generated using `semantic-release`.

Every PR includes a rollback plan, although in practice the rollback mechanism is a `git revert` in the `gitops` repository.

Semantic versioning is used together with immutable image tags that are traceable from the specification all the way to the deployment.

## 13. Multicloud, Disaster Recovery, Chaos Engineering, and FinOps

AWS EKS is the primary cluster and Azure AKS (`aks-dr`) is the secondary cluster used for disaster recovery.

If AWS fails or becomes degraded, traffic can be recovered by serving it from Azure.

The `gitops` repository keeps both clusters synchronized with the same version of every service so that `aks-dr` is always ready to receive traffic without requiring an emergency deployment.

Route 53 handles routing between the two clusters.

Using health checks against each cluster, it can operate in either:

* **Active-passive mode:** all traffic goes to AWS and switches to Azure only if AWS fails.
* **Active-active latency-based mode:** both clusters receive real traffic continuously according to latency, and Azure absorbs the remaining traffic if AWS fails.

Since cost is not a constraint, the documented approach is active-active so that performance between cloud providers can also be compared.

One point must be documented separately: Redis is not replicated between clusters.

Therefore, a failover to `aks-dr` loses the in-memory state of that Redis instance, including pending log queues.

Since business data such as todos and users is also not persisted in a database in the current application, this does not worsen the existing risk, but it should be explicitly stated in the DR plan rather than assuming full data continuity between clouds.

Chaos Mesh executes experiments such as:

* Pod termination
* Network latency
* Redis saturation

These experiments are documented as game days.

They also include simulations of a complete AWS cluster outage to verify that failover to Azure actually works in practice.

OpenCost and Infracost provide cost visibility per service and per infrastructure change.

## 14. AI Agents Repository

Structure of the `ai-agents` repository:

```text
microservice-app-ai-agents/
├── .claude/
│   ├── agents/     # Custom subagents, e.g. terraform-reviewer, incident-responder, spec-writer
│   ├── skills/     # Project slash commands, e.g. /new-feature-spec, /deploy-status, /rollback
│   └── mcp/        # MCP server configuration: GitHub, AWS, Kubernetes, Grafana, ArgoCD
├── docs/           # Documentation explaining the purpose of every agent/skill
└── README.md
```

The adaptation of GitHub Spec Kit as skills also lives here and is made available across all project repositories.

## 15. Spec-Driven Development

The specification is the source of truth.

The code, whether written by a person or an AI agent, is derived from it.

| Stage        | Answers                                                        | Location                                                            |
| ------------ | -------------------------------------------------------------- | ------------------------------------------------------------------- |
| Constitution | How everything in the project must be built                    | `microservice-app-docs`, one shared constitution across the project |
| Specify      | What must be built                                             | `specs/<feature>.md` in the service repository                      |
| Clarify      | Which ambiguities must be resolved before implementation       | Same spec, in the questions and assumptions section                 |
| Plan         | How it will be built                                           | `specs/<feature>/plan.md`                                           |
| Tasks        | Which tasks it is divided into                                 | `specs/<feature>/tasks.md`                                          |
| Implement    | Build according to the specification                           | Normal PR in the service repository                                 |
| Validate     | Verify that the implementation complies with the specification | Contract tests from Section 9                                       |

API contracts are written first:

* OpenAPI for REST APIs
* AsyncAPI for Redis events

CI verifies that the implementation does not deviate from those contracts.

User stories and acceptance criteria move from existing only in the Kanban board to version-controlled files that both the team and AI agents use during implementation.

Not every change requires the full process.

A small fix can go directly to Tasks and Implement.

## 16. Suggested Roadmap

1. Terraform for the three EKS clusters, VPCs, remote backend, and ECR.
2. Platform add-ons: Istio, KEDA, cert-manager, External Secrets, and Kyverno.
3. Scaffold the `gitops` repository, install ArgoCD, and migrate the first service end-to-end.
4. Implement reusable workflows in `.github` and migrate the remaining services.
5. Observability: OpenTelemetry, Jaeger, Prometheus, Grafana, and ELK.
6. Add the Constitution to `docs`, create the first pilot specification, and create the `ai-agents` repository with the Spec Kit skills.
7. Multicloud using AKS and Route 53, Chaos Mesh, and FinOps using OpenCost and Infracost.

## 17. Cost-Optimized Version

Everything described above represents the version without cost restrictions.

This section summarizes the same solution optimized to minimize spending as much as possible without abandoning the engineering practices.

Each reduction replaces a tool with a cheaper equivalent rather than eliminating the practice that the tool supports.

| Aspect                | Full Version                                                                     | Cost-Optimized Version                                                                                                                    |
| --------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Clusters              | 4 (3 EKS + 1 AKS)                                                                | A single EKS cluster, with environments separated by namespaces (`dev`, `staging`, `prod`)                                                |
| Environment isolation | Dedicated VPC and cluster per environment                                        | ResourceQuotas, LimitRanges, NetworkPolicies, and RBAC per namespace                                                                      |
| Multicloud / DR       | Active-active AKS with Route 53                                                  | None. Resilience through multiple AZs within the same cluster; optional cold DR using Velero and S3 snapshots                             |
| Nodes                 | Karpenter, mix of on-demand instances                                            | Karpenter with a high proportion of Spot Instances                                                                                        |
| Service mesh          | Istio with mTLS, traffic shifting, and exact percentage-based canary deployments | No mesh. Resilience patterns are implemented using service libraries; canary deployments use native Argo Rollouts based on replica ratios |
| Logs                  | ELK: Elasticsearch, Logstash, Kibana                                             | Loki, reusing the same Grafana instance used for metrics, with much lower memory consumption                                              |
| Traces                | Jaeger with Elasticsearch backend                                                | Jaeger with embedded storage and short retention                                                                                          |
| Code analysis         | Self-hosted SonarQube                                                            | SonarCloud, free for educational projects, with no self-hosted server required                                                            |
| Image registry        | ECR and mirrored ACR                                                             | ECR only                                                                                                                                  |
| Chaos engineering     | Includes simulation of a complete cloud outage                                   | Only cluster-level experiments such as pod termination, latency, and resource saturation                                                  |

The following remain unchanged in both versions because they do not have significant infrastructure costs:

* GitOps with ArgoCD
* CI with GitHub Actions
* KEDA
* Trivy
* Kyverno
* cert-manager
* External Secrets Operator
* OpenCost
* Infracost
* Kanban
* Trunk-Based Development
* Spec-Driven Development

Environment isolation in the cost-optimized version is weaker than using separate clusters.

A control-plane-level issue or a noisy neighbor on a shared node could affect more than one environment.

The cost-optimized version also gives up the multicloud and service mesh capabilities.

This is a conscious cost decision, not an oversight.
