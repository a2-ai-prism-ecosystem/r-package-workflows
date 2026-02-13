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

## What you copy to your package repository

Copy these files into your package repo under `.github/workflows/`:

- `release.yaml`
- `R-CMD-check.yaml`

`release.yaml` is tag-triggered (`v*`) and intended for release publishing.
`R-CMD-check.yaml` is for more frequent validation so you can catch cross-platform issues earlier than release time.

## Release setup (in your package repo)

1. Copy `release.yaml` into `.github/workflows/release.yaml`.
2. In the `release` job, enable exactly one `uses:` target:
   - `release-gh.yaml` (GitHub release only)
   - `release-gh-prism.yaml` (release + PRISM uploads)
   - `release-gh-prism-edition.yaml` (release + uploads + edition)
3. If your copied file has a local `uses: ./.github/workflows/...` reference, remove/comment it and keep the remote `a2-ai-prism-ecosystem/r-package-workflows/...@v1` reference.
4. Uncomment the `with:` block and enable the platform flags you want for binary builds.
5. Push a tag like `v1.2.3` to trigger the release workflow.

## Required repo configuration

Depending on release mode, configure these in your package repo settings:

- `Repository variable: PRISM_API_URL` (required for API upload/edition flows)
- `Repository secret: PRISM_API_TOKEN` (required for API upload/edition flows)
- `Repository variable: LATEST` (optional, only used by edition flow; set to `true` to mark as latest)

Quick reference:

- `release-gh.yaml`: no PRISM variables/secrets required.
- `release-gh-prism.yaml`: requires `PRISM_API_URL` + `PRISM_API_TOKEN`.
- `release-gh-prism-edition.yaml`: requires `PRISM_API_URL` + `PRISM_API_TOKEN`; optionally uses `LATEST`.

## R-CMD-check usage (recommended frequent validation)

Copy `R-CMD-check.yaml` to `.github/workflows/R-CMD-check.yaml` in your package repo so you can run checks across macOS, Windows, and multiple Linux R versions before tagging a release.

By default this workflow supports manual runs (`workflow_dispatch`) and reusable calls (`workflow_call`). If you want automatic checks on active development, add triggers such as:

- `pull_request`
- `push` to your default branch

This helps answer: "Did I break the package on platforms I am not testing locally?"
