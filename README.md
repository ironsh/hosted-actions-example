# Hosted Actions Example: Using iron-proxy in GitHub Actions

[iron-proxy](https://github.com/ironsh/iron-proxy) is a transparent forward proxy that intercepts all HTTP and HTTPS traffic from your CI job and enforces a domain allowlist. The [`ironsh/iron-proxy-action`](https://github.com/marketplace/actions/iron-proxy) GitHub Action handles all the setup for you. This repository is a working example you can copy and adapt.

## Quick Start

1. Copy [`.github/workflows/ci.yaml`](.github/workflows/ci.yaml) and [`egress-rules.yaml`](egress-rules.yaml) into your repository.
2. Replace the build steps (`npm ci`, `npm test`) with your own.
3. Set `warn: true` on the action and push. The build will pass normally while the proxy logs every outbound request without blocking anything.
4. Check the job summary produced by the `ironsh/iron-proxy-action/summary` step. It shows every domain your build contacted.
5. Add those domains to the `domains` list in `egress-rules.yaml`, remove `warn: true`, and push again. The proxy will now enforce the allowlist.

That's it. Everything below explains what is happening under the hood.

> **Security note:** GitHub Actions gives build jobs `sudo` by default. The action revokes `sudo` for subsequent steps (controlled by the `disable-sudo` input) so that build scripts cannot bypass the proxy. For stronger isolation, we recommend using self-hosted runners in VMs and performing egress enforcement at the hypervisor level.

## Workflow

The workflow uses the [`ironsh/iron-proxy-action`](https://github.com/marketplace/actions/iron-proxy) action, which handles downloading iron-proxy, generating and trusting a TLS interception CA, configuring DNS, starting the proxy, and locking down outbound traffic with iptables — all in a single step:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: ironsh/iron-proxy-action@v0.1.0
        with:
          egress-rules: egress-rules.yaml

      # Your build steps (all traffic goes through iron-proxy)
      - run: npm ci
      - run: npm test

      - uses: ironsh/iron-proxy-action/summary@v0.1.0
        if: always()
```

The `summary` step runs at the end (even on failure) and produces a job summary showing all allowed and denied requests.

### Action Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `egress-rules` | `egress-rules.yaml` | Path to your egress rules file |
| `version` | `latest` | Iron proxy version to install |
| `warn` | `false` | Log denied requests without blocking them |
| `disable-sudo` | `true` | Revoke sudo so subsequent steps can't bypass the proxy |
| `disable-docker` | `true` | Revoke Docker access so subsequent steps can't bypass the proxy |
| `upstream-resolver` | `8.8.8.8:53` | Upstream DNS resolver |

## Configuring the Allowlist

The egress rules live in [`egress-rules.yaml`](egress-rules.yaml):

```yaml
domains:
  # GitHub Actions infrastructure
  - "github.com"
  - "*.github.com"
  - "*.githubusercontent.com"
  - "*.actions.githubusercontent.com"
  - "*.pkg.github.com"
  - "*.blob.core.windows.net"
  - "api.github.com"
  # Stuff your build needs
  - "nodejs.org"
  - "*.nodejs.org"
  - "*.npmjs.org"
```

**`domains`** lists hostnames (with optional wildcards) that are allowed through the proxy. Everything else is blocked.

## How It Works

iron-proxy sits between your CI job and the internet. It has four responsibilities:

1. **DNS interception.** iron-proxy runs a DNS server on `127.0.0.1:53`. When any process resolves a hostname, iron-proxy returns `127.0.0.1`, directing the connection back through itself. It forwards the real lookup to an upstream resolver (`8.8.8.8` by default) to connect to the actual destination.
2. **TLS interception.** For HTTPS, iron-proxy generates certificates on the fly for each destination host, signed by a short-lived CA that the action creates and trusts. Tools like `curl`, `npm`, and `apt` accept these certificates because the CA is in the system trust store.
3. **Allowlist enforcement.** Each request is checked against the domain list in `egress-rules.yaml`. Requests to unlisted hosts are blocked and logged.
4. **Network lockdown.** iptables rules prevent any process from bypassing the proxy by connecting to an external IP directly. Only the proxy and already-established connections are allowed to make outbound connections. All other processes must go through loopback, where the proxy is listening.

## Known Limitations

- GitHub Actions gives build jobs `sudo` by default. The action revokes `sudo` for subsequent steps, but if you set `disable-sudo: false`, attackers who detect the proxy could circumvent it.
- Some runtimes ship their own certificate bundles and may need extra configuration to trust the interception CA. The action handles Node.js (`NODE_EXTRA_CA_CERTS`) automatically; others like Python's `requests` (`REQUESTS_CA_BUNDLE`) or Java (`keytool`) may need manual setup.

## License

This example is provided under the [MIT License](LICENSE).
