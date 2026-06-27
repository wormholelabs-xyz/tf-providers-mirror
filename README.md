# tf-providers-mirror

A Terraform [network mirror](https://developer.hashicorp.com/terraform/cli/config/config-file#network_mirror) for `wormholelabs-xyz`'s private Terraform providers, served from GitHub Pages.

Each provider is built and published to a private GHCR package in its own source repo. This repo collects those artifacts and republishes them as GitHub Releases that the network mirror protocol can serve over HTTPS.

State is declarative: [providers.json](providers.json) is the source of truth. All changes go through pull requests.

---

For operator and consumer details — cutting releases, adding providers, Terraform config — see [USAGE.md](USAGE.md).
