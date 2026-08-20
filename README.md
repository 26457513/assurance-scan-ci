# assurance-scan-ci

Public workflow + template for [assurance-scan](https://github.com/26457513/assurance-scan)
CI scanning. The scanner image (`ghcr.io/26457513/assurance-scan-ci`) is
public glue over open-source scanners (semgrep, trivy, grype, osv-scanner,
syft) — always current at run time.

## Add scanning to a repository

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

If your organisation's Actions policy blocks external workflow references,
use the self-contained copy in [`templates/assurance-scan.yml`](templates/assurance-scan.yml)
instead. For a manual-only variant (zero minutes until triggered from the
UI), delete the `push` and `pull_request` triggers.

Results appear in your repo's Actions summary and PR comments. A deep link
in each report opens the run in an assurance-scan instance when one is
ingesting your organisation (registered via its Settings page).
