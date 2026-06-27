# Usage

## Mirror index

The mirror base URL is `https://wormholelabs-xyz.github.io/tf-providers-mirror/` (no root index — only the provider paths below are served).

Current provider indexes:

| Provider | Index |
|---|---|
| `registrycopy` | [index.json](https://wormholelabs-xyz.github.io/tf-providers-mirror/wormholelabs-xyz.github.io/wormholelabs-xyz/registrycopy/index.json) |
| `tenantaws` | [index.json](https://wormholelabs-xyz.github.io/tf-providers-mirror/wormholelabs-xyz.github.io/wormholelabs-xyz/tenantaws/index.json) |

---

## Consumer configuration

Add this to `~/.terraformrc` (or the file pointed to by `$TF_CLI_CONFIG_FILE`):

```hcl
provider_installation {
  network_mirror {
    url     = "https://wormholelabs-xyz.github.io/tf-providers-mirror/"
    include = ["wormholelabs-xyz.github.io/*/*"]
  }
  direct {
    exclude = ["wormholelabs-xyz.github.io/*/*"]
  }
}
```

Then declare providers using the mirror's host as the source hostname:

```hcl
terraform {
  required_providers {
    registrycopy = {
      source  = "wormholelabs-xyz.github.io/wormholelabs-xyz/registrycopy"
      version = "~> 0.0"
    }
    tenantaws = {
      source  = "wormholelabs-xyz.github.io/wormholelabs-xyz/tenantaws"
      version = "~> 0.0"
    }
  }
}
```

---

## Cutting a release (existing provider)

1. Push a `vX.Y.Z` tag in the provider's private source repo. Its `release` workflow builds the zips, pushes them to GHCR, and prints a ready-to-paste entry in the run summary:
   ```json
   "vX.Y.Z": "sha256:..."
   ```
2. Open a PR adding that line under the matching `<type>` key in [providers.json](providers.json). Keep inner keys sorted by semver. The `validate` workflow asserts the digest against GHCR.
3. Merge. The `mirror` workflow creates the GitHub Release, attaches the zips, and redeploys Pages.

---

## Adding a new provider

1. Create the private source repo (`terraform-provider-<type>` by convention).
2. After the first tag is pushed, the GHCR package exists as private. Grant this repo Actions read access:
   `https://github.com/orgs/wormholelabs-xyz/packages/container/terraform-provider-<type>/settings`
   → **Manage Actions access** → **Add repository** → `wormholelabs-xyz/tf-providers-mirror`, role **Read**.
3. Open a PR adding a new top-level `<type>` key (plus first tag/digest) to [providers.json](providers.json) and merge.
4. Add a row to the table above in this file.

---

## Removing a version

Remove the tag entry from [providers.json](providers.json) and merge. The version disappears from the mirror index; the underlying GitHub Release is left in place so any consumer still pinning it stays unbroken. To hard-delete the release too: `gh release delete <type>/<tag>`.
