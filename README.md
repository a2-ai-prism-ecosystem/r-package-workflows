# r-package-workflows

Reusable GitHub Actions workflows for building, releasing, and deploying R packages with optional PRISM integration.

## R package requirements

In addition to standard R package structure (see [R Packages](https://r-pkgs.org)), these workflows assume your project uses [`rv`](https://a2-ai.github.io/rv-docs/) for dependency management.

**This is important to follow**, as the workflows install dependencies with `rv sync` and pin `rproject.toml` to the target R version during CI.

### Using `rv` in your R package

Keep runtime dependencies in the `DESCRIPTION` file, and use `rproject.toml` for development dependencies.

Your `rproject.toml` should also include a self path dependency (the package under development), so `rv` can correctly resolve and install the package dependency graph.

#### Example `rproject.toml`

```toml
use_lockfile = false

[project]
name = "myRpackage"
r_version = "4.5"

repositories = [
  { alias = "prism", url = "https://prism.a2-ai.cloud/rpkgs/nimbus/2026.01" },
  { alias = "ppm", url = "https://packagemanager.posit.co/cran/latest" },
  { alias = "CRAN", url = "https://cran.r-project.org" },
]

dependencies = [
  "devtools",
  "rextendr",
  # Self-reference to the package being developed
  { name = "myRpackage", path = ".", dependencies_only = false, install_suggestions = true },
]
```

## Quick start

```bash
# 1. Copy the workflow files into your R package repo
mkdir -p .github/workflows

curl -sL https://raw.githubusercontent.com/a2-ai-prism-ecosystem/r-package-workflows/main/.github/workflows/release.yml \
  -o .github/workflows/release.yml

curl -sL https://raw.githubusercontent.com/a2-ai-prism-ecosystem/r-package-workflows/main/.github/workflows/r-release-linux.yml \
  -o .github/workflows/r-release-linux.yml

# Optional: PRISM deployment
curl -sL https://raw.githubusercontent.com/a2-ai-prism-ecosystem/r-package-workflows/main/.github/workflows/deploy-prism.yml \
  -o .github/workflows/deploy-prism.yml

# 2. Search for FIXME and TODO comments and adjust for your package
# 3. Push a tag to trigger the release
git tag v1.0.0
git push origin v1.0.0
# 4. Check the Actions tab and Releases page
```

See the [Cookbook](#cookbook) below for a detailed walkthrough.

## Workflow overview

| File | Purpose | Required? |
| --- | --- | --- |
| `release.yml` | Thin orchestrator — triggers on `v*` tags and calls the build workflow | Yes |
| `r-release-linux.yml` | Builds source tarball + binary packages, publishes a GitHub Release | Yes |
| `deploy-prism.yml` | Deploys release artifacts to a PRISM instance after the release completes | No |

`release.yml` is a short file that you generally don't need to modify. All the build logic and customization points live in `r-release-linux.yml`.

## Cookbook

### Step 1: Copy the workflow files

Download the two required files (and optionally the PRISM deploy file) into `.github/workflows/`:

```bash
mkdir -p .github/workflows

curl -sL https://raw.githubusercontent.com/a2-ai-prism-ecosystem/r-package-workflows/main/.github/workflows/release.yml \
  -o .github/workflows/release.yml

curl -sL https://raw.githubusercontent.com/a2-ai-prism-ecosystem/r-package-workflows/main/.github/workflows/r-release-linux.yml \
  -o .github/workflows/r-release-linux.yml

# Optional — only if you use PRISM
curl -sL https://raw.githubusercontent.com/a2-ai-prism-ecosystem/r-package-workflows/main/.github/workflows/deploy-prism.yml \
  -o .github/workflows/deploy-prism.yml
```

### Step 2: Configure `r-release-linux.yml`

Open `.github/workflows/r-release-linux.yml` and work through the `FIXME` / `TODO` markers.

#### R version

In the `build-primary` job, find the **Setup R** step and set the R version your package targets:

```yaml
      - name: Setup R
        run: |
          dnf install -y curl
          curl -Ls https://github.com/r-lib/rig/releases/download/latest/rig-linux-x86_64-latest.tar.gz | tar xz -C /usr/local
          rig add 4.5              # <-- set your R version
          echo "/opt/R/4.5/bin" >> $GITHUB_PATH   # <-- must match
```

Make sure the **Pin rproject.toml** step further down also matches:

```yaml
      - name: Pin rproject.toml to primary R version
        run: |
          sed -i 's|^r_version = ".*"$|r_version = "4.5"|' rproject.toml   # <-- must match
```

#### Compiled-language toolchain, system dependencies, etc.

If your package uses a compiled language, add the installation steps where the `TODO` blocks appear. There are TODO blocks in `build-primary`, `build-bin-bare`, and `build-bin-container` — add matching steps in each. For example, a Rust-based package would add:

```yaml
      - name: Install Rust
        run: |
          curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
          echo "$HOME/.cargo/bin" >> $GITHUB_PATH
```

Each job also has a `TODO` block for system library installation. Add your package's system dependencies there. For example:

```yaml
      - name: Install system dependencies
        run: |
          dnf install -y epel-release
          MISSING_DEPS="$(rv sysdeps --only-absent) libcurl-devel"
          if [ -n "$MISSING_DEPS" ]; then
            echo "$MISSING_DEPS" | xargs dnf install -y
          fi
```

Keep the same system dependency list in `build-primary` and `build-bin-container`. For `build-bin-bare` (which runs on Ubuntu bare-metal runners), use `apt` package names instead (e.g. `libcurl4-openssl-dev` instead of `libcurl-devel`).

#### Build user

Set the `user` field in the **Build source tarball** step:

```yaml
    user: FIXME   # <-- your build user (e.g. your GitHub org bot username)
```

#### Build options

In the same step, review these flags:

```yaml
    resave-data: false      # true if your package ships data/ that needs resaving
    build-vignettes: false  # true if your package includes vignettes
```

#### Platform matrices

The `plan-matrices` job at the top of the workflow defines two JSON arrays that drive the matrix builds: one for **bare-metal** runners and one for **container** runners.

##### Container matrix entries

Each entry is a JSON object with these fields:

| Field | Description | Example |
| --- | --- | --- |
| `os` | GitHub runner label | `"ubuntu-latest"` |
| `os_name` | Short name for the platform (used in artifact naming) | `"almalinux8"` |
| `arch` | Architecture | `"x86_64"` |
| `r` | R version to install | `"4.4"` |
| `container` | Docker image to run in | `"almalinux:8"` |

Example (the default starter entry):

```json
{"os":"ubuntu-latest", "os_name":"almalinux8", "arch":"x86_64", "r":"4.4", "container":"almalinux:8"}
```

##### Bare-metal matrix entries

Same fields minus `container`:

| Field | Description | Example |
| --- | --- | --- |
| `os` | GitHub runner label | `"ubuntu-24.04"` |
| `os_name` | Short name for the platform | `"ubuntu-noble"` |
| `arch` | Architecture | `"x86_64"` |
| `r` | R version to install | `"4.4"` |

Example:

```json
{"os":"ubuntu-24.04", "os_name":"ubuntu-noble", "arch":"x86_64", "r":"4.4"}
```

The bare-metal array starts empty by default. Add entries as needed.

### Step 3: (Optional) Configure `deploy-prism.yml`

If you use PRISM, open `.github/workflows/deploy-prism.yml` and set the `FIXME` values in the `env:` block:

```yaml
env:
  AUTO_UPDATE_EDITION: 'true'       # Push this version into the shared edition
  AUTO_CREATE_EDITION: 'false'      # Create an individual package edition
  AUTO_LATEST_EDITION: 'false'      # Mark the individual edition as "latest"
  REGISTRY: 'FIXME'                 # <-- your PRISM registry name
  EDITION: 'FIXME'                  # <-- your PRISM edition name (e.g. 'dev', 'stable')
```

Also update the `default:` values for `registry` and `edition` under `workflow_dispatch.inputs` so they match when triggering manually.

#### Required secrets and variables

In your GitHub repo, go to **Settings > Secrets and variables > Actions** and add:

| Type | Name | Description |
| --- | --- | --- |
| Variable | `PRISM_API_URL` | Base URL of your PRISM instance |
| Secret | `PRISM_API_TOKEN` | API token for PRISM authentication |

If you are not using PRISM, no additional secrets or variables are needed — `GITHUB_TOKEN` is provided automatically.

### Step 4: Tag and release

Create and push a tag to trigger the release:

```bash
git tag v1.0.0
git push origin v1.0.0
```

This triggers `release.yml`, which calls `r-release-linux.yml` to build the source tarball, build binaries for each platform in the matrix, and publish everything as a GitHub Release.

Tags ending in `-rc<N>` (e.g. `v1.0.0-rc1`) are published as **pre-releases**.

Both workflows also support `workflow_dispatch` for on-demand runs from the Actions tab.

### Step 5: Verify

1. Check the **Actions** tab for green checks on all jobs.
2. Go to **Releases** and verify the source tarball, binary artifacts, and `manifest.json` are attached.
3. If using PRISM, confirm the package appears in your registry/edition.

## Workflow reference

### `release.yml`

Thin orchestrator that triggers the build workflow. Users generally don't modify this file.

- **Triggers:** `push` on `v*` tags, `workflow_dispatch` with a `tag` input
- **Jobs:** `release-linux` — calls `r-release-linux.yml` via `workflow_call`, passing the tag

### `r-release-linux.yml`

Reusable build workflow (called via `workflow_call`). This is where all customization happens.

#### Triggers

Called by `release.yml` (or any workflow) with a `tag` input via `workflow_call`.

#### Jobs

| Job | Description |
| --- | --- |
| `plan-matrices` | Emits the bare-metal and container JSON arrays that drive the matrix builds |
| `build-primary` | Builds source tarball + one binary on the primary platform (AlmaLinux 8). Publishes both to the GitHub Release. |
| `build-bin-bare` | Matrix job — builds binaries on bare-metal runners. Skipped if the bare-metal array is empty. |
| `build-bin-container` | Matrix job — builds binaries in containers. Skipped if the container array is empty. |
| `finalize-manifest` | Merges all per-build `manifest.json` files into one and attaches it to the release. Runs even if some matrix jobs fail. |

#### Key customization points

| What | Where | Notes |
| --- | --- | --- |
| Primary R version | `build-primary` > Setup R + Pin rproject.toml | Must match in both places |
| Toolchain (Rust, C++, etc.) | TODO blocks in `build-primary`, `build-bin-bare`, `build-bin-container` | Add matching steps in each |
| System deps | TODO blocks in each build job | RPM names for container jobs, deb names for bare-metal |
| Build user | `build-primary` > Build source tarball > `user: FIXME` | Your org bot username |
| Build options | `build-primary` > Build source tarball | `resave-data`, `build-vignettes` |
| Platform matrices | `plan-matrices` job | Two JSON arrays: bare-metal + container |

---

### `deploy-prism.yml`

Deploys release artifacts to a PRISM instance after `R Release Orchestrator` completes.

#### Triggers

| Trigger | Behavior |
| --- | --- |
| `workflow_run` | Auto-triggers after "R Release Orchestrator" completes successfully |
| `workflow_dispatch` | Manual deploy of a specific tag with full control over edition settings |

#### `env:` block (auto-deploy defaults)

| Variable | Default | Description |
| --- | --- | --- |
| `AUTO_UPDATE_EDITION` | `'true'` | Push this package version into the shared edition |
| `AUTO_CREATE_EDITION` | `'false'` | Create a standalone edition scoped to this package |
| `AUTO_LATEST_EDITION` | `'false'` | Mark the individual package edition as "latest" |
| `REGISTRY` | `'FIXME'` | PRISM registry name |
| `EDITION` | `'FIXME'` | PRISM edition name |

#### Jobs

| Job | Description |
| --- | --- |
| `preflight` | Validates PRISM credentials exist. Skips gracefully on auto-trigger; fails on manual dispatch. |
| `resolve-inputs` | Resolves tag, package info, and edition settings from the build manifest or manual inputs. |
| `deploy` | Publishes to PRISM, optionally updates shared edition and/or creates individual package edition. |
