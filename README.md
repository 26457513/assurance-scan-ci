# assurance-scan-ci

Public GitHub Actions distribution for
[Assurance Scan](https://scan.squease.ai).

This repository intentionally contains only:

- the reusable GitHub Actions workflow;
- the small caller template copied into consumer repositories; and
- public documentation for that integration.

The Assurance Scan application repository and deployment configuration remain
private. Runtime images are published separately through GHCR and are
anonymously retrievable so GitHub-hosted runners can use them without package
credentials. As with any public container image, the files shipped inside those
runtime images can be downloaded and inspected.

## Add Assurance Scan to a repository

Use the Setup instructions in Assurance Scan, or copy
[`templates/assurance-scan.yml`](templates/assurance-scan.yml) to
`.github/workflows/assurance-scan.yml` on the repository's default branch.
The public copy uses `main`; replace both trigger values when a repository has a
different default branch. The workflow generated inside Assurance Scan fills in
the selected repository's actual default branch automatically.

The caller deliberately stays small:

```yaml
jobs:
  scan:
    permissions:
      contents: read
      id-token: write
      pull-requests: write
    uses: 26457513/assurance-scan-ci/.github/workflows/scan.yml@main
```

It scans:

- pushes to the default branch; and
- non-draft pull requests targeting the default branch when opened, reopened,
  synchronized or marked ready for review.

Feature branches do not need their own copy of the workflow. Developers can use
the local Assurance Scan container for branches that do not yet have a pull
request targeting the default branch.

## Trust and data flow

The reusable workflow resolves the public producer and uploader images to exact
digests, verifies their Sigstore signatures and verifies the signed release
attestation binding the pair together. It then scans the checked-out revision
on the GitHub runner and sends the bounded result bundle to Assurance Scan using
a short-lived GitHub OIDC token.

No Assurance Scan upload secret is stored in the consumer repository. The full
repository is mounted read-only into the scanner and is not uploaded wholesale.
The upload contains normalized findings, scanner status, bounded code context
around findings, repository/branch/commit provenance, SARIF and the CycloneDX
SBOM.

The server accepts an upload only when GitHub's signed claims identify:

- an enabled GitHub App repository;
- a push to its verified default branch or a pull request targeting it;
- the expected caller path; and
- this reusable workflow on protected `main`.

## Public runtime packages

- `ghcr.io/26457513/assurance-scan-ci`
- `ghcr.io/26457513/assurance-scan-ci-upload`
- `ghcr.io/26457513/assurance-scan-cli`

The hosted application image is not public.
