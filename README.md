# .github

Internal engineering repo for Koa Partners, LLC. Contains organization-wide compliance automation and reusable GitHub Actions workflows consumed by `Koa-Partners/*` repositories.

> **Audience:** Koa Partners engineering only. Not a public profile.

---

## Contents

| File                                                                           | Type                       | Purpose                                                                            |
| ------------------------------------------------------------------------------ | -------------------------- | ---------------------------------------------------------------------------------- |
| [`.github/workflows/audit_mfa.yml`](.github/workflows/audit_mfa.yml)                           | Scheduled (Mon 00:00 UTC)  | Weekly MFA enforcement audit; commits report to `Koa-Partners/compliance-evidence` |
| [`.github/workflows/reusable-codeql.yml`](.github/workflows/reusable-codeql.yml)               | Reusable (`workflow_call`) | CodeQL SAST scan — JS / Python / Go / Java                                         |
| [`.github/workflows/reusable-checkov.yml`](.github/workflows/reusable-checkov.yml)             | Reusable (`workflow_call`) | Checkov IaC scan against SOC 2 framework                                           |
| [`.github/workflows/reusable-gitleaks.yml`](.github/workflows/reusable-gitleaks.yml)           | Reusable (`workflow_call`) | Gitleaks secret scan, full git history; results uploaded to Security tab as SARIF  |
| [`.github/workflows/reusable-license-check.yml`](.github/workflows/reusable-license-check.yml) | Reusable (`workflow_call`) | Allowed-license enforcement for npm / pip / go                                     |

---

## Consuming the reusable workflows

Reference any workflow from a consumer repo with `uses:`. Pin to a tag once we cut releases; until then `@main` is acceptable for internal-only consumption.

### CodeQL SAST

```yaml
jobs:
  codeql:
    uses: Koa-Partners/.github/.github/workflows/reusable-codeql.yml@main
    with:
      language: javascript # javascript | python | go | java
```

### Checkov IaC

```yaml
jobs:
  iac:
    uses: Koa-Partners/.github/.github/workflows/reusable-checkov.yml@main
    with:
      terraform_directory: infra/terraform/ # defaults to terraform/
```

Hard-fails on any finding (`soft_fail: false`). Framework is fixed to `soc2`.

### Gitleaks

```yaml
jobs:
  secrets:
    uses: Koa-Partners/.github/.github/workflows/reusable-gitleaks.yml@main
```

Full-history scan (`fetch-depth: 0`). Runs the Gitleaks CLI directly — no license required. Findings are uploaded to the repo's **Security → Code scanning** tab as SARIF. The job fails if any secrets are detected.

### License Check

```yaml
jobs:
  licenses:
    uses: Koa-Partners/.github/.github/workflows/reusable-license-check.yml@main
    with:
      package_manager: npm # npm | pip | go
```

Allowed licenses (jobs fail on anything outside these sets):

- **npm:** MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, CC0-1.0
- **pip:** MIT, Apache Software License, BSD, ISC License
- **go:** MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC

Need an exception? Raise it before merging the dependency.

---

## Scheduled workflows

### Weekly MFA Audit — `audit_mfa.yml`

Runs Mondays at 00:00 UTC (and on-demand via `workflow_dispatch`):

1. Queries the org for members with MFA disabled.
2. Writes `report.txt`.
3. Commits the report to `Koa-Partners/compliance-evidence` under `automated-evidence/mfa-reports/` as the **Koa Compliance Bot**.

Feeds the SOC 2 access-control evidence trail. Do not disable without coordinating with the compliance owner.

---

## Required secrets

| Secret                | Used by                 | Where it's set | Notes                                               |
| --------------------- | ----------------------- | -------------- | --------------------------------------------------- |
| `ORG_AUDIT_TOKEN`     | `audit_mfa.yml`         | Org-level      | PAT with `read:org`; 90-day rotation                |
| `EVIDENCE_REPO_TOKEN` | `audit_mfa.yml`         | Org-level      | PAT with `contents: write` on `compliance-evidence`; 90-day rotation |
| `GITHUB_TOKEN`        | `reusable-gitleaks.yml` | Auto-provided  | —                                                   |

No license is required for Gitleaks. Consumer workflows do not need `secrets: inherit`.

---

## Changing these workflows

These workflows run against every Koa Partners repo that calls them. Treat changes as production changes:

1. PR against `main`.
2. Validate against at least one consumer repo by pinning `uses:` to your branch (`@your-branch-name`) before merging.
3. Get review from the compliance / platform owner.
4. Communicate breaking input changes in the engineering channel before merge.
