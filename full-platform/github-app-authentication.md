# GitHub App Authentication (spec 009 T103)

## Why two Apps, not one

`.github`'s reusable `release.yml` and `promote.yml` each mint a short-lived
installation token through `actions/create-github-app-token`
(`.github/.github/workflows/release.yml`, `.github/.github/workflows/promote.yml`).
Two separate, single-purpose GitHub Apps back those tokens rather than one
shared App, so that a service repository's release automation can never open a
pull request against `microservice-app-gitops`, and the gitops promoter can
never write to a service repository's own history. No static PAT exists
anywhere in either workflow (`reusable-workflow-contract.bats` asserts this).

| App | Backs | Installed on | Org secrets a caller wires |
| --- | --- | --- | --- |
| `microtodosuite-release` | `release.yml` | The five service repositories only | `RELEASE_APP_ID`, `RELEASE_APP_KEY` |
| `microtodosuite-gitops-promoter` | `promote.yml` | `microservice-app-gitops` only | `GITOPS_PROMOTE_APP_ID`, `GITOPS_PROMOTE_APP_KEY` |

## Exact permissions

Read from what each workflow's minted token is actually used for -- not
requested speculatively:

**`microtodosuite-release`** (`release.yml`):
- Repository permission **Contents: Read and write** -- `actions/checkout` uses
  the token to authenticate, and `semantic-release` pushes a tag and a release
  commit.
- Repository permission **Issues: Write** and **Pull requests: Write** --
  `semantic-release`'s GitHub plugin comments on the issues/PRs a release
  includes.
- No org-level permission. No access to `microservice-app-gitops`,
  `microservice-app-ops`, or `.github`.

**`microtodosuite-gitops-promoter`** (`promote.yml`):
- Repository permission **Contents: Read and write** and **Pull requests:
  Write**, scoped to `microservice-app-gitops` only -- the workflow checks that
  repo out, commits a digest bump on a `promote/...` branch, pushes it, and
  opens the PR (`gh pr create`).
- No permission on any service repository. A compromised or over-broadened
  token from this App still cannot touch service-repo source, CI config, or
  releases.

## Repository installation, not organization-wide

Both Apps are installed with **"Only select repositories"**, not "All
repositories": `microtodosuite-release` on the five service repositories,
`microtodosuite-gitops-promoter` on `microservice-app-gitops` alone. Each
workflow also re-narrows scope at mint time --
`release.yml` passes `repositories: ${{ github.event.repository.name }}` (the
token is valid only for the repository that triggered the run, even though the
App may be installed on all five), and `promote.yml` passes
`repositories: microservice-app-gitops` unconditionally. A token minted for one
service repo's release run is therefore never valid for another service repo,
even though both are covered by the same App installation.

## Selected-repository organization secrets

Both App id/key pairs are organization secrets with **visibility: selected
repositories**, scoped to exactly the five service repositories (the callers):
`RELEASE_APP_ID`, `RELEASE_APP_KEY`, `GITOPS_PROMOTE_APP_ID`,
`GITOPS_PROMOTE_APP_KEY`. `.github`, `microservice-app-gitops`, and
`microservice-app-ops` do not need and do not receive these secrets --
`microservice-app-gitops` is the promoter App's *target*, not a caller of
`promote.yml`.

## Private-key handling: mode 0600, then gone

A GitHub App private key is downloaded once, as a `.pem` file, from the App's
settings page. From that point:

1. `chmod 600` the file immediately, before it is read by anything else.
2. Upload it as the org secret value (`gh secret set GITOPS_PROMOTE_APP_KEY
   --org MicroTodoSuite --repos <the five service repos> < key.pem`, or the
   equivalent for `RELEASE_APP_KEY`) directly from that file -- never paste the
   contents into a terminal, a chat tool, or an intermediate variable that a
   shell history could retain.
3. Delete the local file (`rm -f key.pem`) the moment the secret is confirmed
   set. No copy of a private key lives on a laptop, in a Downloads folder, or
   in any repository -- committed or not.

This mirrors the value-blind discipline `scripts/managed/bootstrap-sonarqube.sh`
applies to the SonarQube administrator credential and analysis token (T103):
a credential is read once, used once, and never echoed or persisted outside
its destination secret store.

## Rotation

GitHub allows an App to hold multiple private keys at once, which makes
rotation zero-downtime:

1. Generate a **new** private key in the App's settings (the old key keeps
   working).
2. Upload the new key as the org secret value, following the mode-0600
   handling above.
3. Trigger one real run of each workflow the App backs (a no-op PR is enough
   for `release.yml`; `workflow_dispatch` or an empty promotion for
   `promote.yml`) and confirm it authenticates.
4. Only after that run succeeds, delete the **old** key from the App's
   settings. Deleting it revokes every token minted from it immediately.

Rotate on any suspected exposure immediately, and otherwise on the same cadence
as the organization's other credential rotation policy.

## Fail-closed authority checks

- **No static credential fallback.** `reusable-workflow-contract.bats` greps
  `release.yml` and `promote.yml` for `aws_access_key_id|AWS_SECRET_ACCESS_KEY
  |AWS_ACCESS_KEY_ID` and for a bare `create-github-app-token@v1` (unpinned) or
  a missing App-id/key secret; any of those fails CI on `.github` itself. A
  workflow edit cannot silently reintroduce a PAT or a long-lived credential.
- **Tuple validation narrows what a valid token can still do.**
  `promote.yml` rejects any `profile`/`destination`/`strategy` combination
  outside the fixed allowlist (`economical/eks-dev`,
  `full/eks-full-{dev,staging,prod}`; `canary` only for `full/eks-full-prod`)
  before the App token is even minted -- an authenticated call with an
  out-of-scope destination still fails closed.
- **Cosign verification gates promotion**, independent of who holds the
  token: `promote.yml` refuses to open a promotion PR for any image whose
  Sigstore keyless signature does not verify against
  `https://github.com/MicroTodoSuite/.+` and the GitHub Actions OIDC issuer,
  so a valid promoter token still cannot promote an unsigned or
  wrongly-attested artifact.
- **No self-approval.** Both Apps only ever open or update a pull request;
  branch protection on every repository requires a named human reviewer, and
  the suite-wide convention (`AGENTS.md` in every repository) forbids
  `--admin` merges, force-pushes to `main`, and an agent approving its own
  pull request. An App token that could technically push does not have a path
  to also merge without a human.
