# shared-workflows

Canonical reusable GitHub Actions workflows for Nish's repos. The CI standard is
**defined once here** and referenced everywhere with `uses:` — updates on the `v1`
branch propagate automatically to every consumer repo. No per-repo sync, no script,
no bot. This is GitHub's own reusable-workflows feature, which is the documented
answer for enforcing a standard across many repos.

## Why this exists

CI must fit inside 3,000 hosted Actions minutes/month against ~2,900 PRs/month —
about one minute of CI per pull request. The standing CI standard
(`nish-vault/_system/shared-memory/global-standing-rules.md` → "CI standard:
batched, minimal, near-zero failures") is enforced here by construction rather
than by anyone remembering to apply it.

## Workflows

### `pr-checks.yml` — reusable PR checks (node/JS + secret scan)

One job, batched: changed-files gate → install → test/build → gitleaks secret scan.
Call it from a consumer repo's `.github/workflows/pr-checks.yml`:

```yaml
name: PR checks
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
  pull-requests: read
jobs:
  pr-checks:
    uses: nish3451/shared-workflows/.github/workflows/pr-checks.yml@v1
    with:
      node-version: "24"
      install-command: npm ci
      verify-command: npm test && npm run build
    secrets: inherit
```

Inputs:

| input | default | purpose |
|---|---|---|
| `node-version` | `24` | Node version for `setup-node`. |
| `install-command` | `npm ci` | Dependency install. Empty string skips install **and** npm cache (for repos with no `package-lock.json`). |
| `verify-command` | `npm test` | Test/build command. Empty string skips verify. Run on code-change PRs only. |
| `python-version` | `""` | Python version for `setup-python`. Empty skips Python setup. |

`secrets: inherit` passes the caller's `GITHUB_TOKEN` through (used by gitleaks
for PR-diff commit ranges).

## How the standard is embodied (by construction)

1. **`timeout-minutes`** on the job — a hang is bounded, not open-ended.
2. **`concurrency` + `cancel-in-progress`** on PR/push only. This workflow only
   runs on PR/push (the caller's trigger); it is never used for
   deploy/release/migration/backup, where cancelling mid-flight is dangerous.
3. **Dependency caching** — `setup-node` `cache: npm` (when a lockfile exists).
4. **Batched** — CI and secret scan are one job, one runner boot, one event.
5. **Run only what changed** — the expensive verify is gated by a cheap changed-files
   check *inside* the job. The job always runs and always reports, so a docs-only PR
   reaches a real green conclusion instead of a skipped required check.
6. **`fetch-depth: 0`** — history is genuinely needed here (the git-diff changed-files
   filter and gitleaks commit scan both need it), which is the documented carve-out.
7. **`retention-days: 7`** on the gitleaks SARIF artifact — it is an ephemeral CI
   byproduct, not durable evidence.
8. **Near-zero failures** — the gitleaks exit-code contract separates "leaks found"
   from "scanner error"; no silent retries.
9. **No scan dropped** — gitleaks runs on **every** PR, including docs-only.
10. **No CI that watches CI** — this workflow only validates the PR in front of it.

## Why gitleaks is the binary, not `gitleaks/gitleaks-action`

`gitleaks/gitleaks-action` requires a `GITLEAKS_LICENSE` secret on **organization**
repos (Nishfleet) and fails with "missing gitleaks license" without it (verified
2026-08-24, run 32697545503). Installing the gitleaks binary directly needs no
license, no secret, and no actions cache — the same shape aiconverter-app and
fleet1-proof-sandbox already run at 0% failure.

## Why not GitHub "required workflows" / repository rulesets

Those would enforce this centrally but are **organization-only**. All five
consumer repos now live in the `Nishfleet` org (where they *could* be used),
but `siterep` was on the personal account `nish3451` when this was designed,
where they cannot — and the design must cover any future personal-account repo
too. Reusable workflows from a public central repo is the one mechanism that
covers **all** repos uniformly, personal and org, current and future. This repo
is public so private consumer repos in any owner can call it.

## Versioning

Consumers pin to `@v1` (the `v1` branch). Improvements land on `v1` and propagate
automatically. A breaking change creates `v2`; consumers opt in by bumping the ref.

This repo is effectively read-only: public so anyone can read it, only Nish can
push. Changes go through PRs reviewed against the standing standard above.

## Adopting a new repo

Use the template repo `nish3451/node-repo-template` ("Use this template") — it is
already wired to `@v1`. Or, in an existing repo, drop in the caller workflow shown
above and delete any per-repo CI/secret-scan workflows it replaces.
