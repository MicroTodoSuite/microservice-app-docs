# Full-Platform Status

Verified state of the MicroTodoSuite platform, its trade-offs, its
data-continuity limits, its traffic status, its rollback path, and the commands
an operator runs to check any of it.

`rollout-index.md` governs stage order and ownership; `plan-reconciliation.md`
records where a specification and reality were reconciled. This document
records only what is true now.

**Verified on 2026-09-20**, from the repositories at `main`. Every claim below
about desired state was read from a file or a render at that point. No claim
about a running cluster is made from this document's authoring environment,
which holds no cluster credentials; the collectors in
`microservice-app-gitops/scripts/managed/` are what produce live evidence, and
today they report `BLOCKED`.

## The single most important fact

**Nothing is deployed.** Every managed cluster registration activates zero
applications, zero infrastructure roots and zero environments:

| Cluster registration | Applications | Infrastructure | Environments |
| --- | ---: | ---: | ---: |
| `eks-dev` (economical) | 0 | 0 | 0 |
| `eks-full-dev` | 0 | 0 | 0 |
| `eks-full-staging` | 0 | 0 | 0 |
| `eks-full-prod` | 0 | 0 | 0 |
| `local-kind` (pilot only) | 0 | 5 | 0 |

The economical runtime is quiesced deliberately, not broken.
`clusters/eks-dev/activation-infrastructure.yaml` states the reason in its own
header: dependency cleanup is complete, so no infrastructure child Application
remains active before a Terraform teardown. The list that was active is
preserved verbatim in `activation-infrastructure-retired.yaml`, so the
retirement is reversible by a reviewed commit rather than by reconstruction.

The three full clusters have never been activated. Their Argo CD bootstrap is
T063-T065, which is open.

A reader should take exactly one conclusion from this table: the platform's
desired state is extensive and tested, and none of it has been observed
running.

## Desired state that exists and is tested

| Capability | Where | Profile |
| --- | --- | --- |
| Tracing to Jaeger over OTLP from all five services | `apps/*/base/configmap.yaml`, `infrastructure/jaeger` | both |
| Technical and business metrics, pull model | `apps/*`, `infrastructure/prometheus` | both |
| Logs to Loki, shipped by Alloy | `infrastructure/loki` (`alloy.yaml`, `alloy-config.yaml`) | economical |
| Logs to Elasticsearch with Filebeat, Logstash, Kibana | `infrastructure/{elasticsearch,filebeat,logstash,kibana}` | full |
| Liveness, readiness and startup probes on every Deployment | `apps/*/base`, `infrastructure/*` | both |
| Alertmanager to Slack, with cluster and environment in the title | `infrastructure/prometheus`, `profiles/full/prometheus/destinations/*` | both |
| Trivy in CI and Trivy Operator in cluster | `.github` `ci.yml`, `infrastructure/trivy-operator` | both |
| Scheduled source, image and cluster assessment with issue routing | `.github` `continuous-security.yml` plus seven callers | both |
| External Secrets against AWS Secrets Manager, with IRSA | `infrastructure/external-secrets` | both |
| RBAC per namespace and service account | `environments/*`, `infrastructure/*` | both |
| Falco, kube-bench and kube-hunter | `infrastructure/{falco,kube-bench,kube-hunter}` | both |
| Digest-only admission through Kyverno | `infrastructure/kyverno`, `profiles/full/kyverno/aws` | both, stricter in full |
| Istio mesh with mTLS and a metric-gated canary | `infrastructure/istio`, `apps/*/components/strategy-canary-full` | full |
| Cost allocation by cluster, namespace, service and profile | `profiles/full/opencost/destinations/*`, `infrastructure/grafana/dashboards/full-profile-cost.yaml` | full |

Each of these renders and is covered by a contract test in
`microservice-app-gitops/tests/`, run by `validate-gitops.yml` on every pull
request. Rendering is not running: see the table above.

## Trade-offs, stated exactly

- **One replica of each platform component**, in both profiles. The full profile
  is not highly available; `009-full-platform-rollout/research.md` Decision 10
  fixes this deliberately, and a reader who assumes redundancy from the word
  "full" would be wrong.
- **The economical profile substitutes**, per the evolution plan section 17:
  Loki and Alloy instead of Elasticsearch, Logstash, Kibana and Filebeat; Jaeger
  with embedded storage and short retention; no Istio, and therefore **no
  internal mTLS**.
- **Alerting is minimal by choice**: one alert per source in the full profile,
  and the upstream node and Kubernetes-state alert sets that ship with
  kube-prometheus are deliberately excluded. Nothing pages on node saturation.
- **node-exporter and kube-state-metrics run only in the full profile**, added
  because OpenCost's metrics reference requires them for cost allocation. The
  economical profile has no node-level or cluster-level metric source, so
  saturation is not observable there.
- **The scheduled security assessment is partly inert.** Its source surface
  works with no credential; its image surface reports `BLOCKED` until a
  read-only ECR role exists; its cluster surface reports `BLOCKED` until a
  cluster carries Argo CD. A blocked surface is never reported as a pass.

## Data-continuity limits

- **Redis is intentionally ephemeral.** Snapshots and append-only persistence
  are off, one replica, per environment
  (`microservice-app-gitops/docs/namespace-isolation.md`). A restart loses
  in-flight messages and any state that had not been consumed. No continuity
  design has been ratified.
- **A lost Redis message is not recoverable and not detected.** The
  log-message-processor consumes Pub/Sub; a message published while it is down
  is gone, and nothing reconciles the gap afterwards.
- **The disaster-recovery continuity claim is `none`.** Spec 009 T138 requires
  that the game day record every sampled message, todo and user as observed,
  lost, duplicated or divergent, and set `durabilityClaim` to `none`. That game
  day has not run.
- **Prometheus, Grafana, Alertmanager and Elasticsearch keep data on encrypted
  volumes in the full profile only.** In the economical profile their storage
  is whatever the cluster gives them, and Alertmanager's silences and
  notification log live in memory.

## Traffic status

- **No public traffic reaches any environment today**, because no environment is
  activated.
- The economical profile declares a frontend ingress
  (`environments/base/ingress-frontend.yaml`) with its own load-balancer network
  policy, contract-tested by `tests/contract/economical-public-entry.sh`. It is
  desired state, not a live endpoint.
- **TLS on ingress and internal mTLS apply to the full profile only.** The
  economical profile has neither, by the section 17 substitutions above.
- **Production routing is not configured.** Route 53 failover between the AWS
  and Azure providers is spec 009 T139 and T140, and T140 may only be applied
  after a separate named human approval of that exact plan and its cost. Until
  then the honest status is `traffic-disabled-ready`.
- Full production declares a metric-gated canary at 10, 25, 50 and 100 percent
  through Istio, with automated analysis. It has never shifted traffic.

## Rollback

- **Every deployment is a commit, and every rollback is a `git revert`.** There
  is no imperative correction path;
  `microservice-app-gitops/tests/policy/no-imperative-managed-mutations.bats`
  fails a pull request that introduces one.
- **The only exception is the audited bootstrap boundary**, documented in
  `microservice-app-gitops/docs/bootstrap-boundary.md`: each new cluster
  receives exactly two audited mutations and nothing else.
- **Retirement is reversible.** The economical activation lists were emptied,
  not deleted; `activation-infrastructure-retired.yaml` holds the previous
  contents, and restoring them is a reviewed commit.
- **Images are pinned by digest**, never by tag, so a revert restores the exact
  artifact that ran before, not whatever a tag points to now.
- A scheduled security assessment cannot break anything to roll back: it runs on
  `schedule` and `workflow_dispatch` only, never on `push` or `pull_request`,
  and holds no `contents: write`.

## Operator commands

Run from the repository root of `microservice-app-gitops`. The first three need
no cluster and no credentials.

```bash
# Render and schema-validate one root exactly as CI does.
kustomize build infrastructure/profiles/full/prometheus/aws \
  | kubeconform -strict -ignore-missing-schemas -summary

# What each destination declares, and what it costs, read-only.
scripts/managed/verify-full-profile-cost.sh      # exit 2 means BLOCKED, not broken

# Multi-cluster desired, live, failure and rollback evidence, read-only.
scripts/managed/verify-full-platform.sh
```

These need a reachable cluster and read-only credentials. They change nothing:

```bash
scripts/managed/verify-observability.sh
scripts/managed/verify-security.sh
scripts/managed/verify-namespace-isolation.sh
```

Every collector reports `PASS`, `FAIL` or `BLOCKED` per check and a final
verdict, and exits 0, 1 or 2 respectively. **A `BLOCKED` verdict is the correct
result today**, not a failure to investigate: it means the check could not
observe what it needed. Treat a `PASS` that should have been `BLOCKED` as the
defect.

To re-read the activation state this document opens with:

```bash
grep -A2 'value:' clusters/*/activation-apps.yaml
```

## What would change this document

| Event | What becomes true |
| --- | --- |
| T063-T065: Argo CD on the three full clusters | The full profile can be activated; the collectors can stop reporting `BLOCKED` |
| T091: activation of the full roots | Desired state becomes running state, per destination |
| A read-only ECR role | The scheduled assessment's image surface starts reporting |
| T135-T141: the DR game day | Continuity is measured rather than declared, and `durabilityClaim` is recorded |
| T139-T140, with separate approval | Production traffic routing moves off `traffic-disabled-ready` |

Until those happen, this document's first table is the state of the platform.
