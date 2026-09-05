# renovate-config

Shared Renovate preset. One file changes the update policy for every repo that
extends it.

## Which preset to use

| Preset | Use it when | Automerge |
|---|---|---|
| `github>noahkiss/renovate-config` | The default branch has branch protection with a required status check | devDependency patch + minor |
| `github>noahkiss/renovate-config:no-automerge` | It does not | none |

Pick by the gate, not by preference. `automerge` with no required check merges
on schedule whether CI passed or not.

## Adopt it

Add `renovate.json` at the repo root.

Repo with a required PR check:

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
| `extends` | `config:recommended` | Dependency dashboard, default grouping and range behaviour |
| `timezone` | `America/New_York` | |
| `schedule` | `* 0-6 * * *` | One overnight review window, so PRs never arrive mid-workday |
| `minimumReleaseAge` | `3 days` | A release must be public three days before it is offered |
| `labels` | `renovate` | |
| `dependencyDashboard` | `true` | |
| `vulnerabilityAlerts.schedule` | `[]` | A CVE fix skips the overnight window |
| `osvVulnerabilityAlerts` | `true` | |

Automerge covers **devDependencies, patch and minor, only**. A devDependency
never reaches the shipped artifact, so the PR's own test run is the whole risk
surface. Production dependencies, majors, base images and GitHub Actions are
merged by hand.

`devDependencies` is the npm depType. Python (pep621) dev groups use different
depType names and stay manual.

## What it deliberately does not set

- **No `prConcurrentLimit`, no `prHourlyLimit`** — and adopting repos should not
  add them. A throttle does not reduce risk; it hides pending updates behind a
  dashboard checkbox. The nightly `schedule` is the only pacing wanted, and
  nothing ships until a PR is merged.
- **No semantic-commit setting** — Renovate detects that per repo from its own
  history. Forcing one answer here would override repos that use conventional
  commits.

## Overriding

A repo's own `renovate.json` is merged after the preset, so its keys win. Repos
with ecosystem-specific rules keep their own config file and add the preset to
their `extends` list rather than replacing it.

## File naming

A bare `github>owner/repo` reference resolves `default.json`, and a named preset
resolves `<name>.json`. Both files here are JSONC — Renovate parses comments in
preset files. Do not rename `default.json` to `default.json5`: it would become
unreachable from the one-line `extends` above.

## Validate a change

```sh
npx --yes renovate-config-validator default.json no-automerge.json
```
