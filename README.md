# assurance-scan-ci

CI security scanning for GitHub repositories — one workflow file, no secrets,
no infrastructure. Scans run on **your** GitHub Actions compute using
always-current open-source scanners; results land in your repo's Actions
summary and PR comments, and optionally in a hosted
[assurance-scan](https://github.com/26457513/assurance-scan) dashboard.

## Architecture

```
your repo ──push/PR──▶ GitHub Actions ──▶ scan.yml
                                            │
                    ┌───────────────────────┘
                    ▼
        docker run ghcr.io/26457513/assurance-scan-ci
        (slim orchestrator, ~150 MB)
                    │
                    ├─▶ semgrep    (code analysis)      ┐ stock public
                    ├─▶ gitleaks   (hardcoded secrets)  │ images, pulled
                    ├─▶ trivy-fs   (dependency CVEs)    │ fresh at run
                    ├─▶ grype      (dependency CVEs)    │ time via the
                    ├─▶ osv-scanner(dependency CVEs)    │ docker socket
                    ├─▶ trivy-config (Dockerfile/IaC)   │
                    ├─▶ syft       (SBOM)               ┘
                    └─▶ trivy-image (if the repo has a Dockerfile)
                                │
                                ▼
            Step Summary · PR comment · SARIF/SBOM/findings artifact
                                │
                    (optional) assurance-scan instance
                    polls your org's runs and mirrors
                    findings into a hosted dashboard
```

The orchestrator image contains only glue code — scanner invocation,
output parsing, report generation. The scanners themselves run as their
stock public images, so rules and vulnerability databases are current at
every run with nothing pinned to maintain.

## Onboarding

### 1. Add the workflow to a repository

Create `.github/workflows/assurance-scan.yml`:

```yaml
name: assurance-scan
on:
  workflow_dispatch:
  pull_request:
    types: [opened, synchronize]
  push:
    branches: [<default branch>]
permissions:
  contents: read
  actions: write
  pull-requests: write
jobs:
  scan:
    uses: 26457513/assurance-scan-ci/.github/workflows/scan.yml@main
```

Replace `<default branch>` (e.g. `main`), commit, push. The next push or PR
runs the first scan. No secrets, no package grants, no other setup.

### 2. Connect a dashboard (optional)

To collect results into a hosted assurance-scan instance — findings
browser, FR catalogues, deep links from PR comments into full reports —
an admin of your organisation:

1. Generates a fine-grained PAT: GitHub → Settings → Developer settings →
   Fine-grained tokens → Generate.
   - **Resource owner**: your organisation.
   - **Repository access**: All repositories.
   - **Permissions**: Contents → Read-only, Actions → Read-only
     (Read **and write** to also enable the dashboard's *Scan now*
     button).
   - If the repository picker is empty: org → Settings → Personal access
     tokens → allow fine-grained tokens, no approval required.
2. Enters the org name and token into the instance's
   **Settings → GitHub organisations**.

The instance verifies the token and begins ingesting scan results within a
minute. Registration can be removed at any time.

### Variants

- **Manual-only** — delete the `pull_request` and `push` triggers; scans
  run only when dispatched from the dashboard or the Actions tab.
- **Restricted Actions policy** — if your org blocks `uses:` references to
  external repositories, use the inlined copy at
  [`templates/assurance-scan.yml`](templates/assurance-scan.yml).

## What each scan produces

| Where | What |
|---|---|
| Run page | Step Summary: per-tool severity matrix with runtimes |
| Pull requests | Findings comment, updated in place per commit |
| Artifact | `assurance-scan-results` — SARIF, CycloneDX SBOM, `findings.json` |
| With a Dockerfile | Additional Trivy scan of the built image |

Scans never fail the workflow; scanner problems appear in the summary.

## Privacy

The workflow runs entirely on your compute and reports only into your
repository. Dashboard integration is opt-in (step 2) and read-only unless
you explicitly grant Actions:Write for the *Scan now* button.
