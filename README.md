# r-package-workflows

Reusable GitHub Actions workflows for building and releasing R packages, with optional PRISM API upload and edition creation.

## What these workflows do

- Build source tarballs for every release.
- Optionally build binaries for selected OS/R-version combinations.
- Optionally upload source/binaries to a PRISM instance.
- Optionally create an individual package edition in PRISM.

Key workflows in this repo:

- `release.yaml`: template workflow you copy into your package repo.
- `release-gh.yaml`: draft release + source/binary artifacts + publish release.
- `release-gh-prism.yaml`: everything in base, plus API uploads.
- `release-gh-prism-edition.yaml`: API uploads + edition creation.
- `R-CMD-check.yaml`: multi-platform package checks for validation between releases.

## Quick start in your package repo

Copy these files into your package repo under `.github/workflows/`:

- `release.yaml` (tag-triggered release workflow, runs on `v*`)
- `R-CMD-check.yaml` (frequent cross-platform validation workflow)

Edit the copied `.github/workflows/release.yaml` in place:

- Keep exactly one active `uses:` line in the `release` job.
- Uncomment `with:` and the build flags you want if you need binaries.
- Leave `secrets: inherit` as-is.
- Push a tag like `v1.2.3` to trigger the workflow.

Example edit in the copied template:

```yaml
jobs:
  release:
    # create release
    # uses: a2-ai-prism-ecosystem/r-package-workflows/.github/workflows/release-gh.yaml@devel
    # create release, build and upload to PRISM
    uses: a2-ai-prism-ecosystem/r-package-workflows/.github/workflows/release-gh-prism.yaml@devel
    # create release, build and upload to PRISM, creating an individual package edition as well
    # uses: a2-ai-prism-ecosystem/r-package-workflows/.github/workflows/release-gh-prism-edition.yaml@devel

    with:
      build-ubuntu-jammy-x86-r45: true
      build-macos-arm-r45: true
    secrets: inherit
```

## Required repo configuration

| Release workflow | `PRISM_API_URL` | `PRISM_API_TOKEN` | `LATEST` |
| --- | --- | --- | --- |
| `release-gh.yaml` | Not required | Not required | Not used |
| `release-gh-prism.yaml` | Required | Required | Not used |
| `release-gh-prism-edition.yaml` | Required | Required | Optional (`true` marks latest) |

## R-CMD-check usage (recommended frequent validation)

Copy `R-CMD-check.yaml` to `.github/workflows/R-CMD-check.yaml` in your package repo so you can run checks across macOS, Windows, and multiple Linux R versions before tagging a release.

By default this workflow supports manual runs (`workflow_dispatch`) and reusable calls (`workflow_call`). If you want automatic checks on active development, add triggers such as:

- `pull_request`
- `push` to your default branch
