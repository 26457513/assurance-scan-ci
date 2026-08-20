# assurance-scan-ci

Public workflow + template for [assurance-scan](https://github.com/26457513/assurance-scan)
CI scanning. The scanner image (`ghcr.io/26457513/assurance-scan-ci`) is
public glue over open-source scanners (semgrep, trivy, grype, osv-scanner,
syft) — always current at run time. All scanning runs on **your** GitHub
compute; nothing reports anywhere else unless you opt in below.

## Onboarding guide (administrator)

Two one-time steps, then one file per repository.

### Step 1 — Register your organisation with an assurance-scan instance (optional)

Do this only if you want scan results collected into a hosted assurance-scan
dashboard (findings browser, FR catalogues, deep links from your PRs). Skip
it to use GitHub-native reports only.

1. Ask the instance operator to open **Settings → GitHub organisations**,
   or do it yourself if you have an account on the instance.
2. Enter your **organisation name** and a **fine-grained personal access
   token** created by an admin of your organisation:
   - GitHub → Settings → Developer settings → Fine-grained tokens →
     Generate new token.
   - **Resource owner**: your organisation.
   - **Repository access**: All repositories.
   - **Permissions**: Actions → **Read-only**, Contents → **Read-only**
     (add Actions → **Read and write** if you also want the UI's
     *Scan now* button to work).
   - Org policy note: if the token picker shows no repositories, your
     organisation must first allow fine-grained PATs (org → Settings →
     Personal access tokens → allow, no approval required).
3. The instance verifies the token and starts ingesting your repos' scan
   results within a minute. Remove the registration at any time from the
   same screen.

### Step 2 — Add the workflow file to a repository

Create `.github/workflows/assurance-scan.yml` in the repository with exactly
this content, replacing `<default branch>` (e.g. `main`) in the `push`
section:

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

Commit and push. The next push to your default branch (or any PR) runs the
first scan — no secrets, no package grants, no other setup.

### Variants

- **Manual-only** (zero minutes until triggered): delete the `pull_request`
  and `push` triggers — scans then happen only via the UI's *Scan now*
  button or a manual workflow dispatch.
- **Blocked external references**: if your organisation's Actions policy
  disallows `uses:` references to outside repositories, use the
  self-contained copy at
  [`templates/assurance-scan.yml`](templates/assurance-scan.yml) instead —
  same behaviour, inlined steps.

## What each scan produces

- A **Step Summary** on the run page: per-tool severity matrix with
  runtimes, plus any scanner failures.
- A **PR comment** (on pull requests) with the same summary, updated in
  place per commit.
- An **`assurance-scan-results` artifact**: SARIF findings, a CycloneDX
  SBOM, and the normalized `findings.json`.
- Repos with a root `Dockerfile` additionally get a Trivy image scan of
  the built image.

Scans never fail your workflow — scanner problems are listed in the summary.

## Scanner set

| Tier | Scanners |
|---|---|
| Always | semgrep (code), gitleaks (secrets), trivy-fs + grype + osv-scanner (dependencies), trivy-config (IaC/Dockerfile), syft (SBOM) |
| With a Dockerfile | trivy-image (built image) |

All run as their stock public images — current rules and vulnerability
databases at run time, with no pinned versions to maintain.
