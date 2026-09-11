# Branching Strategy

**Status**: current. The binding rules are in
[Pull request and task tracking conventions](Pull%20request%20and%20task%20tracking%20conventions.md);
this page summarizes the model and records what it replaced.

## The model

Every MicroTodoSuite repository uses trunk-based development with short-lived
branches.

![Trunk-based development](./assets/Trunk-Based%20Development.png)

- `main` is the trunk. It is protected and always releasable.
- Work happens on one short-lived branch per concern, named
  `<type>/<short-kebab-summary>` — for example `feat/full-profile-secrets` or
  `fix/bump-image-digest-only`.
- Every change reaches `main` through a pull request that follows the
  repository's template and passes its checks.
- There are no long-lived `develop`, `release`, or environment branches.
  Environments are promoted by pull requests that change desired state in
  `microservice-app-gitops`, never by merging one branch into another.
- Incomplete capability stays off by default. Full-profile switches in the
  infrastructure roots, for example, default to disabled until their stage is
  approved.

## Commits and releases

- Commits follow Conventional Commits with a scope. The commit type drives the
  released version.
- A change developed test-first lands as two commits, a failing
  `test(<scope>): specify ...` commit followed by `feat(<scope>): implement ...`.
  The pair is never squashed.
- Service repositories and `microservice-app-ops` release with semantic-release,
  which tags `vX.Y.Z` and writes the changelog.
- The Terraform module repositories release each module independently with
  release-please, under tags of the form `<module>-vX.Y.Z`.
- Image promotion in `microservice-app-gitops` uses `promote/*` branches that
  change only an image digest.

## Review and merge

`microservice-app-gitops` requires one approval on `main`. In every other
repository the author merges a green pull request, with the exceptions listed in
§6 of the conventions. No one approves their own pull request, and an AI agent
never approves one.

## What this replaced

Until August 2026 the application repositories followed GitHub Flow and the
operations repository committed directly to `main`. The two models converged
into the single model above when the conventions were adopted; GitHub Flow with
short-lived feature branches is the same practice under another name, and direct
commits to `main` are no longer permitted anywhere.
