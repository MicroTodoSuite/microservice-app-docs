# Governance and IaC Standards Program

**Date**: 2026-09-10
**Owner**: the maintainer (Esteban Gaviria)
**Status**: plan. Section 11 lists the decisions it waits on.

On 2026-09-10 the maintainer asked for nine things at once:

1. Parameterize everything that should be parameterized.
2. An infrastructure-as-code rule set in the AI repository, adapted from the
   course rules and extended to every provider and resource the project uses,
   including how everything is parameterized and named.
3. An audit of the current code against that rule set.
4. Everything in the organization in English, professional, and in the third
   person.
5. A recommendation on how to separate the IaC repositories.
6. Rules for how agents install, connect to, and use MCP servers.
7. Rules for commits and pull requests, including one that makes an agent merge
   its own green pull request where no approval is required.
8. A project board that reflects the real, current work.
9. Outdated documentation from before the Kubernetes and multi-cloud migration
   corrected.

This document inventories what exists, answers the design questions, and orders
the work end to end. It is itself written to rule 4.

---

## 0. Done on 2026-09-10

| Item | Where |
| --- | --- |
| Rule 4 recorded: English covers every artifact in the organization; voice is professional and third person | `docs#19`, conventions §3 |
| Rule 7's merge clause recorded: the author merges a green pull request where no approval is required, with three stated exceptions | `docs#19`, conventions §6 and §9 |
| The AWS account declared once and moved with one command | `microservice-app-ops#33` (held — see below), `microservice-app-gitops#88` (awaits approval), `.github#13` and five service pull requests (await the organization variable) |
| Capacity, regions, and the `demo-full` runbook | `docs#18` |

`microservice-app-ops#33` is green but held under the new §6: the in-progress
`feat/profile-lifecycle` work edits one of its files.

---

## 1. What exists today

| Area | State | Evidence |
| --- | --- | --- |
| Commit and pull-request rules | Written and binding, but **not enforced by any tool** | `docs/Pull request and task tracking conventions.md`, ten PR templates, every `AGENTS.md`; no commitlint, PR-title check, or branch-name check in any repository; semantic-release in seven |
| AI repository | Private, 47 files: Spec Kit skills, `.specify/`, four scripts. No IaC, MCP, or git rules, no skill beyond Spec Kit | `microservice-app-ai-agents` on `main` |
| AI repository README | Advertises `.claude/agents/` (terraform-reviewer, incident-responder, spec-writer) and `.claude/mcp/`, which exist on **no branch** | `git ls-tree` across every remote branch |
| Rule distribution | Other repositories symlink to the AI repository and to this repository; the links are dangling wherever the siblings are not checked out, including CI | `.agents/skills`, `.claude/skills`, `.specify/*` in ten repositories |
| MCP servers | Only in each person's user configuration. `codebase-memory-mcp` is documented; no Terraform, AWS, Azure, GitHub, or Kubernetes server is configured or documented anywhere | user and Codex configurations; no `.mcp.json` in any repository |
| Project boards | Projects 4 and 5, twelve items each, every one a 2025 Azure Container Apps pull request plus one issue. None of the 186 open tasks appears | `gh project item-list` |
| Board templates | Spanish (`Kanban - Equipo de Desarrollo`) | `.github/projects/` |
| Branch protection | Only `microservice-app-gitops` requires approval. The ops ruleset requiring one approval is **disabled**. The AI repository cannot be protected on the Free plan while private | branch protection and rulesets APIs |
| Language | Spanish in the organization profile, three repository descriptions, `.github/projects/`, and four documents here; seven repositories have no description at all | see §7 |
| Obsolete repositories | `microservice-app-prometheus` (a standalone Prometheus image, superseded by the Prometheus Operator in GitOps and referenced nowhere) and `microservice-app-example` (the 2025 upstream application) | repository contents and references |
| Course rules | 26 files, `PC-IAC-001` to `PC-IAC-026`, Spanish, dated 2025-12-10 and 2025-12-11, AWS-centric, outside every repository in `MicroTodoSuite/rules-iac-modules/` | local folder, extracted from a macOS archive |

---

## 2. Preliminary conformance — a first measurement, not the audit

A grep-level scan of `microservice-app-ops` at `f3a5ea5` against the course
rules. It sizes the audit; it does not replace it.

| Rule | Finding |
| --- | --- |
| PC-IAC-001 module structure | All five modules lack `.gitignore`, `CHANGELOG.md`, `README.md`, `data.tf`, `locals.tf`, `providers.tf`, and `sample/`; `environment-foundation` also lacks `main.tf` and spreads its resources across sixteen files |
| PC-IAC-002 global variables | No module declares `client`; two declare `project` and `environment` |
| PC-IAC-003 naming | Live names carry no `client` segment and several exceed 28 characters: `microtodosuite-github-ecr-publisher` (35), `microtodosuite-tfstate-575172595729-us-east-1-dev` (49) |
| PC-IAC-005 provider aliases | No `principal` alias, no `configuration_aliases`, no `provider = aws.project` anywhere |
| PC-IAC-010 collections | 43 `count` against 18 `for_each`; 9 `depends_on` |
| PC-IAC-011 data sources | 15 data sources inside modules; the rule allows them in roots only |
| PC-IAC-015 module consumption | 9 local `../` sources; no remote, SemVer-tagged source |
| PC-IAC-018 static analysis | No tflint, Checkov, or Trivy configuration |
| PC-IAC-023 single responsibility | 13 IAM roles created inside modules, and 5 calls to upstream VPC and EKS modules from inside `environment-foundation`, which spans the networking, security, and workload domains at once |
| PC-IAC-004 tags | `default_tags` present in six provider blocks; `Owner` and `CostCenter` appear |
| PC-IAC-019 remote state | Not used — compliant |

The conclusion matters for scheduling: the largest gaps are **structural** —
module layout, module consumption, domain separation — not cosmetic. They are
answered by §3, not by a formatting pass.

---

## 3. Repository separation

### One repository per profile? No.

The economical and full profiles are environments of one platform:

1. **They share account-level resources.** The state backend, the ECR
   repositories, the GitHub OIDC provider, the Route 53 zone, and the secret
   containers are owned by the `dev` root and read by the full-profile roots.
   Splitting by profile turns those into cross-repository dependencies.
2. **They use the same modules.** Per-profile repositories would duplicate the
   modules or reach into each other for them.
3. **The course rules separate along other axes.** PC-IAC-015 separates
   *modules from roots* — "one module, one repository", consumed by SemVer tag.
   PC-IAC-022 separates *domains inside a root* — networking, then security,
   then workload. What differs between the profiles is configuration, and
   PC-IAC-024 assigns configuration to per-environment `.tfvars`.

### Recommended layout

| Kind | Repositories | Contents |
| --- | --- | --- |
| Reference modules | One per module, named `terraform-<provider>-<name>` (decision G4) | The PC-IAC-001 layout with `sample/`, `CHANGELOG.md`, tests; released by SemVer tag |
| AWS live configuration | `microservice-app-ops` | `shared/` for account-level resources; `environments/<environment>/{networking,security,workload}/` per PC-IAC-022; both profiles as environments; every module consumed by tag |
| Azure live configuration | Same repository under `azure/`, or `microservice-app-ops-azure` (decision G5) | The AKS disaster-recovery estate |
| Kubernetes desired state | `microservice-app-gitops`, unchanged | |

Candidate modules once `environment-foundation` is decomposed under PC-IAC-023:
networking (VPC, subnets, NAT), transit egress, EKS cluster, EKS node capacity,
IRSA role (security domain), ECR repository, Secrets Manager secret, Route 53
zone, and state backend; for Azure, AKS cluster, virtual network, Key Vault, and
ACR. The decomposition design is the first deliverable of phase P6, not a
decision taken here.

### Constraints on the migration

- **Live resources must not be recreated.** The economical platform is the
  rollback target, and its refreshed plan must stay `0/0/0`. Extraction into
  modules uses `moved` blocks and state moves, proven by that plan.
- **The full profile is not applied yet**, so it can adopt the new layout
  directly, at no migration cost.
- **`feat/profile-lifecycle` restructures the same roots.** The separation starts
  after it lands, or that work adopts the layout — decision G8.

---

## 4. The rule set

### Where it lives

In `microservice-app-ai-agents`, which becomes the single place agents read
rules from:

```
rules/
  README.md          index and precedence: constitution > conventions > rules > skills
  language.md        pointer to conventions §3
  git.md             operational summary of the conventions; the conventions stay the source
  iac/
    PC-IAC-001.md … PC-IAC-026.md   the course rules, adapted to English, IDs kept
    MTS-IAC-1xx.md                   project additions for providers the course rules do not cover
  mcp.md             section 5
skills/              iac-author, iac-review, iac-audit, delivery
mcp/                 committed MCP configuration, distributed to every repository
```

### Format of every rule

ID; title; source (the course rule and section it adapts); the requirement in
RFC 2119 form; rationale; what it applies to; the automated check that enforces
it (a tflint rule, a Checkov or Trivy ID, or a contract script); the agent
procedure — which MCP server or official page to consult, and what evidence goes
into the pull request; and how an exception is recorded.

### Additions the course rules do not cover

The course rules are written for AWS alone. The project also needs:

- **Azure** — naming with Cloud Adoption Framework abbreviations and each
  service's length and character limits (a storage account accepts no hyphens
  and 24 characters at most); provider aliases mirroring PC-IAC-005; an Azure
  Blob backend with lease locking, which PC-IAC-008's "S3 only" has to allow.
- **Kubernetes desired state** — images by digest, no imperative mutation of
  GitOps-managed state; the existing GitOps rules, restated as rules.
- **GitHub Actions as infrastructure** — actions pinned by full SHA, OIDC
  instead of static keys.
- **Account and region as declared parameters** — the contract introduced by
  `microservice-app-ops#33`.
- **Upstream registry modules** such as `terraform-aws-modules` — exact version
  pins and how they relate to PC-IAC-015.

### Conflicts to resolve, not paper over

| Course rule | Conflict | Proposed adaptation |
| --- | --- | --- |
| PC-IAC-003 | `{client}-{project}-{environment}-{type}-{key}` in 28 characters; the project has no `client` and uses `microtodosuite` (14 characters) | Decision G2 |
| PC-IAC-003 | S3 bucket names are global, so the state bucket carries the account ID and exceeds 28 | A recorded service-constraint exception |
| PC-IAC-002 | Environments `dev`, `qa`, `pdn`; the project has `dev`, `staging`, `prod`, `demo` in two profiles | Decision G2 |
| PC-IAC-003 and PC-IAC-025 | Renaming live resources replaces them — an EKS cluster, an IAM role in use, a bucket holding state | Decision G3 |
| PC-IAC-008 | S3 is "the only supported backend"; the Azure estate uses Azure Blob | Extend to one backend per cloud |
| PC-IAC-011 | Data sources only in roots; some module logic reads caller identity or partition | Enumerate the allowed read-only data sources |

### Enforcement

- A reusable `iac-checks` workflow in `.github`, called by every IaC repository:
  `terraform fmt`, `validate`, and `test`; tflint with the AWS and AzureRM
  rulesets; Checkov or Trivy configuration scanning; and contract scripts for
  structure, naming, tags, and remote module sources.
- Four skills for agents: **iac-author** (write Terraform to the rules, consulting
  the MCP servers), **iac-review** (review a diff against them), **iac-audit**
  (produce a findings register), and **delivery** (branch, commit, open, verify,
  and merge a pull request exactly as the conventions say, including §6).
- The `AGENTS.md` generator adds a "Rules that apply to this repository"
  section to each repository's file.

---

## 5. MCP servers

### Which, and for what

| Server | Used for |
| --- | --- |
| `codebase-memory-mcp` | Cross-repository code graph; already documented |
| HashiCorp Terraform MCP server | Registry, provider, and module documentation |
| AWS documentation or knowledge MCP server | Service limits, naming constraints, recommended practice |
| AWS API MCP server, **read-only** | Discovery against the account |
| Azure MCP server, **read-only**, and Microsoft Learn documentation MCP server | The same for Azure |
| GitHub MCP server, or the `gh` CLI | Issues, boards, pull requests |

Exact package names and versions are taken from each vendor's documentation at
implementation time and pinned. Nothing here is installed from memory.

### Rules

- MCP configuration MUST come from a file committed in the AI repository and
  distributed, not from each person's memory: `.mcp.json` for Claude Code, and a
  documented block for Codex, which reads user scope.
- Versions MUST be pinned. Configuration MUST NOT contain secrets; it references
  environment variables.
- Cloud servers MUST run read-only under least-privilege identities. An agent
  MUST NOT change cloud or cluster state through an MCP server; changes go
  through saved Terraform plans and GitOps.
- Before writing a provider argument, a version, or a limit, an agent SHOULD
  consult the documentation server and cite what it checked in the pull request.
  If a server is unavailable, the agent says so and uses the vendor's official
  documentation online, never memory alone.
- Installation and verification are documented in `docs/mcp.md` in the AI
  repository and checked by a script.

---

## 6. Commits and pull requests

The conventions document stays the single source. What is missing is
enforcement:

- A reusable `conventions` workflow in `.github`, called by every repository,
  that fails a pull request whose title is not a scoped Conventional Commit, whose
  branch is not `<type>/<kebab-summary>`, or whose template sections are missing
  or empty.
- Repository settings, applied by an organization owner: squash merge only, the
  pull-request title as the squash commit title, and head branches deleted on
  merge.
- The **delivery** skill from §4, so that agents follow the rules by procedure,
  not only by instruction.
- Whether to enable the disabled ops ruleset is decision G6. Enabling it means
  agents stop merging in ops under §6.

---

## 7. English and voice

| Artifact | Action |
| --- | --- |
| `.github/profile/README.md` | Rewrite for the current architecture |
| `.github/projects/` | Replace with an English board definition, or remove with the old boards (decision G7) |
| Repository descriptions | English descriptions for all twelve repositories; seven have none |
| `README.md` here | Rewrite as an index |
| `docs/Agile methodology.md` | Rewrite for the process and board actually used |
| `docs/Branching strategies.md` | Replace with a short page pointing at the conventions; its GitHub Flow and trunk-based split no longer describes practice |
| `docs/Architecture diagrams.md` | Rewrite for EKS, AKS, and GitOps; Azure Container Apps was retired on 2026-08-30 |
| `docs/Report.md` | Delete; it is empty |
| English documents and agent rules | A third-person pass; this program's own recent documents included |
| `microservice-app-ops/AGENTS.md` | Remove the two bullets that still describe the retired Azure roots |
| `microservice-app-prometheus`, `microservice-app-example` | Archive (decision G7) |

Vendored upstream text — Spec Kit's skills, vendored Kubernetes manifests — is
treated as vendored and not rewritten, by the same reasoning that exempts
vendored manifests from other rules. That is an assumption the maintainer can
overturn.

---

## 8. The project board

A single English organization project replaces the two 2025 boards, which are
closed and kept as history rather than deleted.

| Field | Values |
| --- | --- |
| Status | Backlog, Ready, In Progress, In Review, Blocked, Done |
| Lane | Infrastructure, CI/CD, Observability, Governance |
| Gate | Gate 0, Gate 1, Gate 2, per the execution plan |
| Spec and Task ID | For example `gitops 009 T052` |
| Size | S, M, L |

Items are GitHub issues generated from the task registers with the Spec Kit
`taskstoissues` skill, one parent issue per block with the tasks as sub-issues
(granularity is decision G1). The registers remain the source of truth; the board
is a view of them. A pull request closes its issue with `Closes #n` and still
updates `tasks.md`, and an issue closed while its task is unticked is a defect.
The built-in project workflows move closed items to Done.

---

## 9. Parameterization

| What | Where it is literal today | Target |
| --- | --- | --- |
| AWS account | — | **Done**, pending merges |
| Region | 414 occurrences in ops, 62 in GitOps; hard-pinned validations in the full-profile roots | One input per environment, as `capacity-and-regions.md` requires |
| Resource names | `microtodosuite` 238 times in ops, 529 in GitOps | The PC-IAC-003 and PC-IAC-025 naming, built once in each root |
| GitHub organization | 97 times in ops, in OIDC subjects | An input |
| Domain | 36 times in ops | Per-environment `.tfvars`, which partly exists |
| Terraform version | 17 times in ops, 3 in GitOps | One source, `.terraform-version` |
| CIDRs, availability zones, sizes | Hard-pinned validations and defaults | Per-environment `.tfvars` under PC-IAC-024 |

It runs after the rules (P4), because the rules decide how names and variables
are built, and after `feat/profile-lifecycle`, because it touches the same
roots. Kubernetes manifests keep account and region as literals by design —
ArgoCD renders plain Git — each with one declared source and a contract.

---

## 10. Phases and order

| Phase | Deliverables | Depends on | Lane |
| --- | --- | --- | --- |
| P1 | The decisions in §11 | — | Maintainer |
| P2 | §7: language and voice, repository descriptions, obsolete repositories | P1 (G7) | Governance |
| P3 | AI repository restructure, rule skeleton, MCP rules and configuration, delivery skill, conventions workflow | — | Governance |
| P4 | The 26 adapted rules, the additions, the conflict resolutions, the `iac-checks` workflow | P1 (G2, G3) | Infrastructure |
| P5 | The audit: every IaC repository against every rule, as a findings register with tasks | P4 | Infrastructure |
| P6 | Repository separation: design, module repositories from a template, `environment-foundation` decomposed, domain layout | P5, `feat/profile-lifecycle` (G8) | Infrastructure |
| P7 | Remediation and §9's parameterization | P6 | Infrastructure |
| P8 | The new board, populated from the registers | P1 (G1) | Governance |

The critical path is P1 → P4 → P5 → P6 → P7. P2, P3, and P8 run alongside it.

Under §7 rule 5 of the conventions, this program needs a task register before
its work starts. The proposal is a Spec Kit feature in the AI repository,
`specs/001-governance-and-iac-standards`, generated from this document, so that
every row above carries a task ID.

---

## 11. Decisions — taken by the maintainer on 2026-09-11

| # | Decision | Consequence |
| --- | --- | --- |
| G1 | The board groups work into **large functional blocks**: one issue per block, with a checklist of its activities; never an issue per task. **All 2026 work appears**, completed included, consolidated in blocks | The registers stay the task-level record; the board is the block-level view |
| G2 | A client is defined, and the naming convention is formal and documented | Client `gcs` (GaCode Solutions, the owner named in the organization profile), project `mts`, environments `shd`, `eco`, `fdev`, `fstg`, `fprd` — `rules/iac/MTS-IAC-101` in the AI repository |
| G3 | **Adopt the convention fully, now**: bring down every deployed resource that must change, rename, and recreate it — EKS included — after preserving everything persistent or sensitive, then validate and leave no orphan or old reference | ops spec 004; constitution amendment for the economical rebuild; MTS-IAC-107 |
| G4 | **One module repository per cloud provider** (`terraform-aws-modules`, `terraform-azure-modules`), not one per module and not one for all; modules versioned independently | `rules/iac/MTS-IAC-102`; supersedes PC-IAC-015's "one module, one repository" |
| G5 | Re-evaluate why the project is multicloud and redefine the criterion if it is weak, keeping multicloud as the goal | `docs/ADR-0001 Multicloud strategy.md`: Azure becomes an independent recovery domain, active-passive |
| G6 | Requiring one approval on the ops `main` branch is the **definitive policy**, but it stays **disabled** until the pending implementation is complete and stable | A board item holds the activation, triggered by the end of the rebuild's validation |
| G7 | Keep `microservice-app-prometheus` if it is still used; archive `microservice-app-example` if it serves no function; archive the old boards | `example` archived on 2026-09-11 (an unreferenced 2025 fork). `prometheus` is referenced nowhere, and the economical cluster runs the Prometheus Operator from GitOps, not this image — recommended for archiving, awaiting the maintainer's confirmation |
| G8 | Land `feat/profile-lifecycle` first, refactor afterwards; never in the same change | Landed as ops#34 and #35 |
| G9 | `MicroTodoSuite/rules-iac-modules/` is the source of the rules | Reviewed in full and adapted |
| — | Create and use `AWS_ACCOUNT_ID` properly | Declared per MTS-IAC-103; the organization variable needs an organization owner's token scope |

Two decisions change earlier constraints and therefore need recorded
amendments rather than this table alone: G3 overrides the rule that the
economical environment is never destroyed, and G5 rewrites constitution
principle 12. Both are in the constitution amendment that accompanies
ADR-0001.
