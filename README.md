# renovate-config

Shared Renovate preset. One file changes the update policy for every repo that
extends it.

## Which preset to use

| Preset | Use it when | Automerge |
|---|---|---|
| `github>noahkiss/renovate-config` | A workflow runs on `pull_request` | devDependency patch + minor |
| `github>noahkiss/renovate-config:no-automerge` | No CI runs on pull requests | none |

Branch protection is not required. The gate is Renovate's own, not GitHub's;
see "How the automerge gate works" below.

## Adopt it

Add `renovate.json` at the repo root.

Repo with CI on pull requests:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>noahkiss/renovate-config"]
}
```

Repo without one:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>noahkiss/renovate-config:no-automerge"]
}
```

Then add the repo in the Renovate app's install settings.

## What the default preset sets

| Key | Value | Why |
|---|---|---|
| `extends` | `config:recommended`, `:group-non-major` | Dependency dashboard and range behaviour; non-major grouping (below) |
| `timezone` | `America/New_York` | |
| `schedule` | `* 0-6 * * *` | One overnight review window, so PRs never arrive mid-workday |
| `minimumReleaseAge` | `3 days` | A release must be public three days before it is offered |
| `labels` | `renovate` | |
| `dependencyDashboard` | `true` | |
| `prHourlyLimit`, `prConcurrentLimit` | `0` | No throttle. Renovate's own defaults are 2 an hour and 10 open, so omitting the keys is not enough. Adopting repos should not set them: a throttle hides pending updates; the `schedule` is the only pacing |
| `rebaseWhen` | `conflicted` | A branch is rebased only when it conflicts. Each rebase is a push, and each push runs the PR's CI again |
| `vulnerabilityAlerts.schedule` | `[]` | A CVE fix skips the overnight window |
| `vulnerabilityAlerts.groupName` | `null` | A CVE fix is its own PR, never part of a group |
| `osvVulnerabilityAlerts` | `true` | |

## Grouping

`group-non-major.json` folds non-major updates into at most two PRs per repo per
night. Every PR runs the repo's CI, once per push to its branch, and private
repos pay for those minutes.

| Group (branch) | Takes | Automerge |
|---|---|---|
| `non-major dependencies` (`renovate/non-major`) | Every minor, patch and digest update the other group does not take | No |
| `non-major devDependencies` (`renovate/non-major-dev`) | npm devDependency minor and patch updates | Yes, by the rule below, which matches exactly this group |

Majors are not grouped: each stays its own PR. A vulnerability fix is never
grouped. A repo's own `groupName` rules come after the preset's, so they still
win for the packages they match.

A repo whose tooling needs one PR per dependency opts out:

```json
{
  "extends": ["github>noahkiss/renovate-config"],
  "ignorePresets": ["github>noahkiss/renovate-config:group-non-major"]
}
```

`noahkiss/infra` does: its triage job merges and deploys one compose path per PR.

## Automerge

Automerge covers **devDependencies, patch and minor, only**. A devDependency
never reaches the shipped artifact, so the PR's own test run is the whole risk
surface. Production dependencies, majors, base images and GitHub Actions are
merged by hand.

`devDependencies` is the npm depType. Python (pep621) dev groups use different
depType names and stay manual.

## How the automerge gate works

The automerge rule sets `platformAutomerge: false`. Renovate does the merge
itself on a later run, and only when the PR branch is green:

- Every commit status has passed and every check run concluded `success`,
  `neutral` or `skipped`.
- A pending or in-progress check is not green. Renovate waits for the next run.
- A failed check is red. Renovate leaves the PR open and adds reviewers.
- No checks at all is not green. Renovate does not automerge a branch with no
  status checks unless `ignoreTests` is `true`, and this preset never sets it.
- Renovate's own `renovate/stability-days` status does not count toward green
  by default, so a repo whose only check is Renovate's never merges.

`minimumReleaseAge` adds that `renovate/stability-days` status as pending until
the release is three days old, so the PR cannot merge before then either.

The alternative, `platformAutomerge: true`, hands the merge to GitHub's
auto-merge, which waits only for **required** checks. Branch protection and
rulesets are not available on private repos on GitHub's free plan, so nothing
there can be required and GitHub would merge before CI finished. Renovate's
docs say the same: without a required check the platform "might merge Renovate
PRs even if the repository's tests haven't started, are still in progress, or
possibly even when they have failed". The Renovate-side gate has no such
dependency.

A repo with no CI on pull requests never goes green, so the rule would not
merge there either. It still extends `:no-automerge` so the intent is explicit
and its PRs do not advertise "Automerge: Enabled".

Expect a delay. The Mend hosted app runs a repo's job every few hours plus on
webhook events, and Renovate merges at most one PR per base branch per run.

## What it deliberately does not set

- **No semantic-commit setting** — Renovate detects that per repo from its own
  history. Forcing one answer here would override repos that use conventional
  commits.

## Overriding

A repo's own `renovate.json` is merged after the preset, so its keys win. Repos
with ecosystem-specific rules keep their own config file and add the preset to
their `extends` list rather than replacing it.

## File naming

A bare `github>owner/repo` reference resolves `default.json`, and a named preset
resolves `<name>.json`. Every file here is JSONC — Renovate parses comments in
preset files. Do not rename `default.json` to `default.json5`: it would become
unreachable from the one-line `extends` above.

## Validate a change

```sh
npx --yes --package renovate -- renovate-config-validator default.json no-automerge.json group-non-major.json
```
