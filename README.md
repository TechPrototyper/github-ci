# github-ci

Shared, reusable CI pipeline for the TechPrototyper account. One
`workflow_call` workflow that any repo can call with a ~15-line wrapper.

## What it does

A single `ci.yml` reusable workflow with four jobs:

- **lint** — language-aware:
  - python → `ruff`
  - go → `golangci-lint`
  - javascript → `eslint`
- **tests** — python (`pytest`) and go (`go test ./...`), auto-skips if no
  test suite is detected.
- **security** — all OSS, runs offline, no paid API needed:
  - `gitleaks` — secret / credential leak detection (drop-in replacement
    for GitHub Secret Scanning).
  - `semgrep` with the `p/default` OSS rule pack — SAST; only HIGH/CRITICAL
    findings fail the job (low/info are reported in the log, not fatal).
  - dependency audit — `pip-audit` (python), `govulncheck` (go),
    `npm audit` (javascript), against the OSV/CVE databases.
- **report** — on failure:
  - if the run came from a **pull request** → posts a PR comment.
  - if the run came from a **push** → opens (or dedupes) an Issue tagged
    `ci-failure`, so nothing is silently lost.

On green it does nothing (no noise).

## How to wire it into a repo

Copy `templates/ci-wrapper.yml` into the target repo as
`.github/workflows/ci.yml`, then edit three things:

1. `branches: [ main ]` → your default branch
2. `language:` → `python` / `go` / `javascript` / `shell` / `none`
3. `python-version:` → if python

Then commit. The reusable workflow does the rest. No secrets are needed —
`GITHUB_TOKEN` (automatic) covers PR comments and Issue creation.

`secrets: inherit` is required so the reusable run can call the GitHub API.

## Security-only repos

For repos that have no meaningful test/lint target (docs, Verilog,
shell-only), set `language: none` or `shell` and keep `run-security: true`.
You get gitleaks + semgrep for free, no lint/test noise.

## Dependencies / token budget

- Runs on the free-tier minutes (public: unlimited; private: 2,000/min/mo).
- A typical run is ~2–4 minutes. ~30 repos × 2 runs/day ≈ 120 min/month ≈ 6%.
- No paid tools: every scanner here is OSS (gitleaks, semgrep OSS rules,
  pip-audit/govulncheck/npm-audit, ruff, golangci-lint, eslint).
- Semgrep OSS rule pack is bundled and runs offline — no `SEMGREP_APPSEMGREP_TOKEN`
  needed.

## Files

- `.github/workflows/ci.yml` — the reusable workflow (source of truth)
- `.github/workflows/self-test.yml` — `workflow_dispatch` self-test
- `templates/ci-wrapper.yml` — per-repo wrapper to copy
- `templates/dependabot.yml` — Dependabot config to copy
