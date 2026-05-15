# tf-providers-mirror

Terraform network-mirror host for `wormholelabs-xyz`'s private Terraform
providers. Each provider lives in its own private source repo and publishes
build artifacts to its own private GHCR package; this repo collects them
into plain GitHub Releases that the Terraform network mirror protocol
serves from GitHub Pages.

State is declarative — [providers.json](providers.json) is the source of
truth — and pinned by OCI manifest digest, so even if an upstream tag is
retroactively repushed, the mirror still serves the artifact that was
approved.

No PATs, no GitHub Apps. The cross-repo hop is each upstream package
granting *this* repo Actions read access; the mirror's own
`GITHUB_TOKEN` then authenticates the `oras pull`.

## Flow

```
[private source repo: terraform-provider-<type>]
    │  push tag vX.Y.Z
    ▼
  release.yml: goreleaser build → ORAS push
                                       │
                                       │  digest emitted in run summary
                                       ▼
   ghcr.io/wormholelabs-xyz/terraform-provider-<type>:vX.Y.Z
        (private, Actions read access granted to this repo)
                                       │
                                       │  PR adding "vX.Y.Z": "sha256:..."
                                       │  to providers.json; validate.yml
                                       │  asserts digest matches; merge.
                                       ▼
[this repo: wormholelabs-xyz/tf-providers-mirror]
  mirror.yml:
    reconcile     # for each declared (type, tag, digest):
                  # if release missing, oras pull by digest +
                  # create release with assets
    build index   # generate static network-mirror JSON
    deploy-pages  # publish to Pages
```

## Cutting a release

1. Push a `vX.Y.Z` tag in the provider's private source repo. The source
   repo's `release` workflow builds the zips, pushes them to its private
   GHCR package, and prints a ready-to-paste providers.json entry in the
   run summary.
2. Open a PR here adding that entry under the corresponding `<type>` key in
   [providers.json](providers.json). The `validate` workflow checks the
   digest against GHCR. Merge when green.
3. The `mirror` workflow creates the release, attaches the zips, and
   redeploys Pages.

## providers.json

```json
{
  "registrycopy": {
    "v0.0.1": "sha256:abcd1234...",
    "v0.0.2": "sha256:ef567890..."
  },
  "another": {
    "v1.0.0": "sha256:09abcdef..."
  }
}
```

- **Top-level keys** are provider types. Each becomes a directory under
  the network-mirror layout and resolves a GHCR package at
  `ghcr.io/wormholelabs-xyz/terraform-provider-<type>`.
- **Inner keys** are git-tag strings (with the leading `v`). The
  corresponding release in this repo is tagged `<type>/<tag>`.
- **Values** are the OCI manifest digests of the GHCR artifact for that
  tag. Reconcile pulls by digest, so even if the tag drifts on GHCR, the
  mirror serves the artifact that was approved at PR-merge time.
- **Adding** a tag: edit, open PR, merge. `validate` checks the digest;
  `mirror` creates the release.
- **Removing** a tag: edit, merge. The version disappears from the
  network-mirror index; the release object is left in place so consumers
  who still pin it stay unbroken. To hard-delete, also `gh release delete
  <type>/<tag>`.

## Adding a new provider

Per provider, once:

1. Create the private source repo (named `terraform-provider-<type>` by
   convention; the source `release.yml` derives the GHCR target from the
   repo name).
2. After the first tag push, the GHCR package will exist as **private**.
   Grant this repo Actions read access:
   `https://github.com/orgs/wormholelabs-xyz/packages/container/terraform-provider-<type>/settings`
   → "Manage Actions access" → "Add repository" → select
   `wormholelabs-xyz/tf-providers-mirror`, role **Read**.
3. Open a PR adding the new `<type>` key (plus first tag/digest) to
   [providers.json](providers.json) and merge.

## One-time setup for this repo

1. **Enable GitHub Pages**: Settings → Pages → Source: "GitHub Actions".

## Consumer configuration

```hcl
# ~/.terraformrc or $TF_CLI_CONFIG_FILE
provider_installation {
  network_mirror {
    url = "https://wormholelabs-xyz.github.io/tf-providers-mirror/"
  }
  direct {
    exclude = ["wormholelabs-xyz.github.io/*/*"]
  }
}
```

```hcl
terraform {
  required_providers {
    registrycopy = {
      source  = "wormholelabs-xyz.github.io/wormholelabs-xyz/registrycopy"
      version = "~> 0.1"
    }
    # other providers from this mirror use the same host/namespace pattern
  }
}
```

The provider's source address (`wormholelabs-xyz.github.io/wormholelabs-xyz/<type>`)
is just an identifier — Terraform resolves it via the network mirror URL,
which is this repo's Pages root.
