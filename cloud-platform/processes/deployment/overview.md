# Cloud Platform ERP — Deployment & CI/CD Overview

This document gives an overview of the current CI/CD setup for the Cloud Platform ERP
(`jtl-platform-erp`). Note: Other Cloud Platform repos has the similar if not the same structure.

It is built on GitHub Actions, deploying to Azure (AKS + Helm +
Terraform + ACR), and splits cleanly into **caller workflows** (triggered by events) and
**reusable workflows** (the shared building blocks).

## The architecture at a glance

```
PR opened ──► pr-check ──────────► (lint, test, build, infra-validate) gate merge
PR labeled "deploy" ─► pr-deploy ─► ephemeral PR environment
PR closed/unlabeled ─► pr-cleanup ─► tear down PR env
                       pr-ttl-cleanup (nightly cron) ─► reap stale PR envs

push main ──────► release-dev ──► tag → build → release(dev) → e2e
push release ───► release-semver ─► tag → merge into release-qa
push release-qa ─► release-qa ───► tag → build → release(qa) → e2e + load-test
push release-beta ► release-beta ─► tag → build → release(beta)
push release-prod ► release-prod ─► tag → build → release(prod)

All of the above call the shared building blocks:
  workflow-build · workflow-test · workflow-infra-validate · workflow-release · workflow-e2e-test · workflow-loadtest
```

## The promotion pipeline (environments)

The core flow is a branch-driven promotion ladder, each branch mapped to an environment:

| Branch | Environment | Versioning | Extra steps after release |
|--------|-------------|-----------|--------------------------|
| `main` | **dev** | hash-based tag | E2E tests |
| `release` | (semver) | semantic `-pre` tag, then **auto-merges into `release-qa`** | — |
| `release-qa` | **qa** | semantic `-qa` tag | E2E + **load tests** |
| `release-beta` | **beta** | semantic `-beta` tag | — |
| `release-prod` | **prod** | semantic `v` tag | — |

Versioning uses `wemogy/get-release-version-action`. Note the inconsistency: dev is
hash-based, while qa/beta/prod are semantic with environment-specific suffixes, and only
`release-semver` does the auto-merge promotion into QA.

## The reusable building blocks

- **`workflow-build`** — builds 1 frontend (`app`) + 4 backend Docker images (`api-main`,
  `background`, `api-admin`, `migration`) via a matrix, exports them as `.tar` artifacts.
  Layer caching keyed on `pnpm-lock.yaml` / `*.csproj`.
- **`workflow-test`** — frontend unit + coverage (Vitest), frontend Docker-script tests,
  backend tests with coverage.
- **`workflow-infra-validate`** — `terraform validate` + `plan` (PR comment) and
  `helm lint` + `helm diff`. Read-only preview; uses the **PR** Azure identity.
- **`workflow-release`** — the real deploy: `terraform apply` → push images to ACR →
  `helm upgrade`. Uses the **deploy** Azure identity on a self-hosted
  `Azure-Private-Network-Runner`.
- **`workflow-e2e-test`** — Playwright against the deployed env, publishes HTML report to
  `gh-pages` (keeps last 5 runs).
- **`workflow-loadtest`** — Azure Load Testing + Locust, pulls a test wheel from JFrog;
  scenarios: baseline/stress/soak/spike.

## PR experience

- **`pr-check`** runs on every PR to main: commitlint, prettier, eslint, Danger, plus
  build/test/infra-validate gated by a `paths-filter` (only runs the parts that changed).
  A final `status-check` job aggregates everything into one required check.
- **`pr-deploy`** is opt-in via a `deploy` label — spins up a full ephemeral PR
  environment at `<pr#>.pr.erp.dev.jtl-cloud.com` with its own Terraform + Helm release.
- **`pr-cleanup`** + **`pr-ttl-cleanup`** (nightly cron) tear those down.

## Operational utilities (manual / scheduled)

- **`restore-data`** / **`restore-data-scheduler-qa`** — CosmosDB restore (manual, plus
  monthly cron for QA).
- **`switch-connection`** — connection switching (reusable).
- **`terraform-force-unlock`** — manual recovery from stuck Terraform state locks.
- **`tf-plan-dev`** — Terraform plan on PRs.

## Candidate pain points

Things that already stand out as candidates for the problem statement:

- The build → push-as-`.tar` artifact → re-import → push-to-ACR indirection.
- Per-environment branch sprawl with inconsistent versioning.
- The `gh-pages` force-push for E2E reports.
- Heavy reliance on a single self-hosted runner (`Azure-Private-Network-Runner`).
- Terraform `apply` running inline with deploys.
