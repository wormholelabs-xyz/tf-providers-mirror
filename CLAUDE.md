# CLAUDE.md — tf-providers-mirror

## Project Overview

This repo is the Terraform network-mirror host for `wormholelabs-xyz`'s
private Terraform providers. It collects build artifacts from per-provider
private GHCR packages and republishes them as plain GitHub Releases that
Terraform's network-mirror protocol can serve over HTTPS from GitHub Pages.

Each upstream provider lives in its own private source repo
(`terraform-provider-<type>`) that pushes builds to its own private GHCR
package; this repo is granted Actions read access on each package and
republishes the zips on merges to main.

This repo holds **no provider code**. It's a state machine: a declarative
config ([providers.json](providers.json)) plus two workflows that
reconcile state against GHCR + GitHub Releases + Pages. There is nothing to
build locally and nothing to run outside CI.

## Architecture

```
providers.json         → declarative state (source of truth)
.github/workflows/
  mirror.yml           → reconcile (pull missing from GHCR, create releases)
                         + build mirror index + deploy Pages
  validate.yml         → assert each declared digest still resolves at GHCR
```

All workflow logic is inlined with `actions/github-script` — no checked-in
shell scripts. Octokit handles JSON manipulation, API calls, and binary
asset uploads natively. The one shell-out is `oras pull` / `oras resolve`,
which is unavoidable.

## What's where

- [providers.json](providers.json) — `{ "<type>": { "<tag>": "<sha256-digest>" } }`.
  Source of truth for everything served.
- [.github/workflows/mirror.yml](.github/workflows/mirror.yml) — two-job
  workflow on push-to-main: `reconcile` (pulls from GHCR by digest, creates
  releases) + `deploy-pages` (builds network-mirror index, deploys).
- [.github/workflows/validate.yml](.github/workflows/validate.yml) — runs
  on PRs touching `providers.json` and on main pushes. For each declared
  `(type, tag, digest)`, asserts `oras resolve` against GHCR returns the
  declared digest.

## Key Decisions

These were made with reasoning; don't reverse without re-reasoning.

### Declarative state in `providers.json`

State lives in a file, not in tags or release metadata. Adding/removing
versions is a PR, with full diff and review. The mirror workflow is a
reconciler — running it twice is a no-op.

The alternative (workflow inputs each release, no checked-in config) was
the first cut and got rejected: you lose the audit trail and the
diff-driven review, and there's no obvious answer to "what's currently
served." With `providers.json`, that question has a one-file answer.

### Pull by digest, not tag

`providers.json` pins each tag to an OCI manifest digest. The reconcile
workflow pulls `<repo>@<digest>`, not `<repo>:<tag>`. If an upstream tag
is retroactively repushed, this mirror still serves the artifact approved
at PR-merge time. The validate workflow asserts the declared digest still
matches the tag on GHCR — that's a sanity check (catches typos and
upstream drift), not a security control. Security comes from the
digest-pinned pull.

### GHCR packages stay private; mirror is granted Actions access

Per-package "Manage Actions access" grants this repo's `GITHUB_TOKEN` read
access to each upstream package. No public packages, no PATs, no GitHub
Apps. The mirror workflow does an `oras login ghcr.io` using its own
`GITHUB_TOKEN` and authenticated `oras pull` works.

The original draft made the upstream package public to avoid any
authenticated cross-repo hop. The granted-access model is just as
PAT-free, and keeps the upstream artifacts private — closer to least
privilege.

### `actions/github-script`, not shell scripts

CI-only code, so the JS-with-octokit ergonomics (native JSON, typed API
client, real error handling, binary upload via Buffer) win cleanly over
bash portability or local-runnability. Don't reintroduce
`scripts/<thing>.sh` here.

### Single repo for all providers

One mirror serves N providers; `providers.json` is a map keyed by type.
Releases here are tagged `<type>/<tag>` (slashes are valid in git tag
names) so types don't collide. The index builder writes one provider
directory per top-level key.

Adding a new provider is additive: new private source repo, grant package
access, add a key to `providers.json`. No code changes here.

### Soft delete on tag removal

Removing a tag from `providers.json` removes it from the network-mirror
index; the underlying GitHub Release object is *not* deleted. Consumers
that still pin the removed version stay unbroken until their state moves
forward. Hard delete is explicit: `gh release delete <type>/<tag>`.

This was deliberate. The whole point of pinning is reproducibility;
silently breaking an old pin is the opposite of what consumers want.

### Single workflow handles reconcile + Pages

`mirror.yml` has two jobs (`reconcile` → `deploy-pages`). They could
trivially be two separate workflows chained via `workflow_run`, but
`workflow_run` doesn't fire reliably for events triggered by the default
`GITHUB_TOKEN` (GitHub suppresses them to prevent recursion). Keeping
both jobs in one workflow sidesteps the issue and is simpler.

## Conventions

- **No checked-in shell scripts.** All workflow logic is inlined via
  `actions/github-script`. The one shell-out is `oras`.
- **Pinned action SHAs.** Every `uses:` line includes a SHA pin and a
  trailing `# <version>` comment.
- **Concurrency group `mirror`** on the mirror workflow — no overlapping
  reconciles.
- **`providers.json` keys sorted alphabetically** by provider type and by
  tag (semver) within each provider. Easier diffs.

## Adding a release

For an existing provider:

1. Upstream pushes `vX.Y.Z`. The source repo's `release` workflow run
   summary contains a ready-to-paste line like:
   ```json
   "vX.Y.Z": "sha256:abc..."
   ```
2. Open a PR adding that line under the matching `<type>` key in
   [providers.json](providers.json). Keep the inner keys sorted.
3. The `validate` workflow asserts the digest matches what's at GHCR.
4. On merge, `mirror` reconciles (creates the release + uploads zips +
   deploys Pages).

## Adding a new provider

See [README.md](README.md). Summary:

1. Private `terraform-provider-<type>` repo is created upstream.
2. First tag is pushed; upstream's GHCR package now exists (private).
3. Grant this repo Actions read access on that package via the package's
   "Manage Actions access" UI.
4. PR a new top-level key in `providers.json` with the first tag/digest.
