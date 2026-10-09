# Pipeline Analysis — Designing Safe Deployment

> **Branch:** `fix/pipeline` | **Author:** Samragyi Sharma | **Date:** 2026-10-09

---

## 1. Broken Pipeline — Original File

```yaml
# .github/workflows/deployment.yml (BROKEN)
name: Deploy Orion API

on:
  push:
    branches:
      - '*'   # ← triggers on ALL branches

jobs:
  deploy:                          # ← runs FIRST — directly to production
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '16'
      - run: npm install           # ← should be npm ci
      - run: npm run build         # ← no artifact saved
      - run: bash scripts/deploy.sh # ← to PRODUCTION, no gates
      - run: curl ... || echo "... continuing anyway"  # ← failure silenced
      - if: always()
        run: echo "status ${{ job.status }}"  # ← not a real notification

  lint:
    needs: deploy                  # ← AFTER deploy — lint is useless here
    ...
  test:
    needs: lint                    # ← AFTER deploy — tests are useless here
    ...
```

---

## 2. Problem Inventory

### 2.1 Missing Validation Stages

| # | Missing Stage | Impact |
|---|--------------|--------|
| 1 | **Lint** runs AFTER deploy | Code style/syntax errors reach production before being caught |
| 2 | **Tests** run AFTER deploy | Broken logic ships to production; failures are post-mortem, not preventive |
| 3 | **Security scan** is entirely absent | Vulnerable dependencies and leaked secrets can ship unchecked |
| 4 | **Coverage gate** does not exist | Tests may pass with 0% coverage; no minimum threshold enforced |

### 2.2 Incorrect Execution Order

The broken pipeline runs stages in this order:

```
deploy (production) → lint → test
```

Correct order must be:

```
source → build → lint → test → security → deploy-staging → [approval] → deploy-production → verify
```

**Root cause:** `lint` and `test` both use `needs: deploy` / `needs: lint`, meaning they are *downstream* of production deployment — completely backwards.

### 2.3 Absent Safety Gates

| Gate | Present in Broken Pipeline? | Risk |
|------|-----------------------------|------|
| Branch filter (only `main`/`fix/*` trigger prod) | ❌ No — `branches: ['*']` | Feature branches accidentally deploy to production |
| Staging environment before production | ❌ No staging job exists | No pre-production validation environment |
| Manual approval before production | ❌ No approval gate | Any push auto-deploys to production |
| Test pass required before deploy | ❌ Tests run after deploy | Failed tests do not block the release |
| Security scan gate | ❌ Completely absent | High/critical CVEs ship to production |
| Coverage threshold | ❌ Completely absent | Untested code ships |

### 2.4 Failure Isolation Problems

- All steps live inside **one monolithic job** (`deploy`). When a step fails, the error message appears in a wall of logs with no labelled job boundary.
- The smoke test uses `|| echo "... continuing anyway"` — **failures are swallowed**. The pipeline reports success even when the service is down.
- There is no `timeout-minutes` set — a hung deployment can block the runner indefinitely.
- No `id` is assigned to steps, making it impossible to reference specific step outcomes.
- No structured traceability: commit SHA, timestamp, and deployer identity are not logged.

### 2.5 Rollback Gaps

- `rollback.sh` exists on disk but is **never wired into the pipeline**.
- No previous image tag is captured or stored as an artifact — even if rollback were triggered, there is nothing to roll back to automatically.
- The `verify` / smoke-test failure does not trigger any automated rollback; operators would have to intervene manually with no documented procedure.
- No `on-failure` step calls `rollback.sh`.

---

## 3. Designed Fix — Stage Map

```
Push to fix/pipeline or main
        │
        ▼
┌───────────────┐
│   source      │  Checkout + env metadata
└──────┬────────┘
       │ needs: source
       ▼
┌───────────────┐
│   build       │  npm ci → lint → build → upload artifact
└──────┬────────┘
       │ needs: build (build must succeed)
       ▼
┌───────────────┐
│   test        │  Download artifact → jest --coverage → coverage gate ≥ 80%
└──────┬────────┘
       │ needs: test
       ▼
┌───────────────┐
│   security    │  npm audit --audit-level=high + secret scan
└──────┬────────┘
       │ needs: security  (only on main/fix/pipeline)
       ▼
┌───────────────────┐
│  deploy-staging   │  Simulated staging deploy + healthcheck
└──────┬────────────┘
       │ needs: deploy-staging  (only on main — MANUAL APPROVAL)
       ▼
┌──────────────────────┐
│  deploy-production   │  environment: production (required reviewers)
└──────┬───────────────┘
       │ needs: deploy-production
       ▼
┌───────────────┐
│    verify     │  Smoke tests → on failure → rollback trigger
└───────────────┘
```

### Gate Conditions

| Stage | Gate Condition |
|-------|---------------|
| build | Exit 0 on lint AND build |
| test | All jest tests pass + coverage ≥ 80% |
| security | `npm audit` no high/critical + no secret patterns found |
| deploy-staging | All prior jobs succeeded |
| deploy-production | Staging verified + manual approval via GitHub Environment |
| verify | HTTP 200 from `/health`; failure triggers `rollback.sh` |

---

## 4. Artifact Strategy

| Artifact | Uploaded by | Downloaded by | Purpose |
|----------|-------------|---------------|---------|
| `build-artifact` | `build` job | `test`, `deploy-staging`, `deploy-production` | Guarantees the exact same compiled output flows through every stage; no re-building with different deps |

---

## 5. Key Fixes Applied

| Problem | Fix Applied |
|---------|------------|
| `branches: ['*']` | Changed to `push: branches: [main, 'fix/**']`; production deploy only on `main` |
| `npm install` | Changed to `npm ci` (deterministic, reproducible) |
| Lint/test after deploy | Moved to BEFORE deploy using `needs` chain |
| No security scan | Added `npm audit --audit-level=high` in dedicated `security` job |
| No staging environment | Added `deploy-staging` job with simulated staging deploy and healthcheck |
| No approval gate | Added `environment: production` with GitHub required reviewers |
| Smoke test failure silenced | Removed `|| echo`; curl failure now exits non-zero and fails the job |
| No rollback | `verify` job calls `scripts/rollback.sh` on failure via `if: failure()` step |
| No traceability | Each job logs `COMMIT_SHA`, `RUN_ID`, `TIMESTAMP`, and `GITHUB_ACTOR` |
| No failure notification | Added `notify` job with `if: failure()` to report which job failed |
| No coverage gate | Jest configured with `--coverage --coverageThreshold` (80% lines) |
