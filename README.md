# Hosted Actions Example: Using iron-proxy in GitHub Actions

[iron-proxy](https://github.com/ironsh/iron-proxy) is a transparent forward proxy that intercepts all HTTP and HTTPS traffic from your CI job and enforces a domain allowlist. This repository is a working example you can copy and adapt.

## Quick Start

1. Copy [`.github/workflows/ci.yaml`](.github/workflows/ci.yaml) and [`iron-proxy.yaml`](iron-proxy.yaml) into your repository.
2. Replace the commented build steps (`npm ci`, `npm test`) with your own.
3. Push and let the workflow run. It will probably fail because your build contacts hosts that aren't in the allowlist yet.
4. Check the "Print proxy log" step in the workflow output. It shows every request iron-proxy handled, including blocked ones.
5. Add the blocked domains to the `domains` list in `iron-proxy.yaml`, commit, and re-run. Repeat until your build passes.

That's it. Everything below explains what is happening under the hood.

## Configuring the Allowlist

The proxy configuration lives in [`iron-proxy.yaml`](iron-proxy.yaml). The key section is the `transforms` block:

```hosted-actions-example/iron-proxy.yaml#L15-30
transforms:
  - name: allowlist
    config:
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
        - "registry.npmjs.org"
        - "*.npmjs.org"
```

**`domains`** lists hostnames (with optional wildcards) that are allowed through the proxy. Everything else is blocked.

## Viewing the Proxy Log

The workflow includes a final step that always runs, even if earlier steps fail:

```hosted-actions-example/.github/workflows/ci.yaml#L82-87
- name: Print proxy log
  if: always()
  run: |
    echo "=== Proxy log ==="
    cat /var/log/iron-proxy.log | grep --line-buffered '^{' | \
      jq -r '[(.time | split(".")[0]), .audit.action, .audit.host, .audit.method, .audit.path] | @tsv'
```

iron-proxy writes structured JSON logs. This command extracts the timestamp, action (`allow` or `deny`), host, HTTP method, and path into a readable table. When something is blocked, this log is the fastest way to find out which domain you need to add.

---

## How It Works

iron-proxy sits between your CI job and the internet. It has four responsibilities:

1. **DNS interception.** iron-proxy runs a DNS server on `127.0.0.1:53`. When any process resolves a hostname, iron-proxy returns `127.0.0.1`, directing the connection back through itself. It forwards the real lookup to an upstream resolver (`8.8.8.8` by default) to connect to the actual destination.
2. **TLS interception.** For HTTPS, iron-proxy generates certificates on the fly for each destination host, signed by a short-lived CA that the workflow creates and trusts. Tools like `curl`, `npm`, and `apt` accept these certificates because the CA is in the system trust store.
3. **Allowlist enforcement.** Each request is checked against the domain and CIDR lists in `iron-proxy.yaml`. Requests to unlisted hosts are blocked and logged.
4. **Network lockdown.** iptables rules prevent any process from bypassing the proxy by connecting to an external IP directly. Only root (the user iron-proxy runs as) and already-established connections are allowed to make outbound connections. All other processes must go through loopback, where the proxy is listening.

## Detailed Walkthrough

The entire proxy setup happens inside a single `run:` block in the workflow. Here is each piece, in order.

### Download and Install iron-proxy

```hosted-actions-example/.github/workflows/ci.yaml#L18-25
# Download and install iron-proxy
echo "=== Installing iron-proxy ==="
export VERSION=0.4.0
curl -fsSL -o /tmp/iron-proxy.tgz \
  https://github.com/ironsh/iron-proxy/releases/download/v${VERSION}/iron-proxy_${VERSION}_linux_amd64.tar.gz
tar -xzf /tmp/iron-proxy.tgz -C /tmp
sudo mv /tmp/iron-proxy /usr/local/bin/iron-proxy
sudo chmod +x /usr/local/bin/iron-proxy
```

Downloads a pinned release of iron-proxy and places it on the `PATH`.

### Generate a CA for TLS Interception

```hosted-actions-example/.github/workflows/ci.yaml#L27-37
# Generate a CA for TLS interception. Must have keyUsage=critical,keyCertSign and CA constraints
echo "=== Generating CA for TLS interception ==="
mkdir -p /tmp/iron-proxy-ca
openssl genrsa -out /tmp/iron-proxy-ca/ca.key 2048 2>/dev/null
openssl req -x509 -new -nodes \
  -key /tmp/iron-proxy-ca/ca.key \
  -sha256 -days 1 \
  -subj "/CN=iron-proxy CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign" \
  -out /tmp/iron-proxy-ca/ca.crt 2>/dev/null
```

iron-proxy needs a CA certificate and key to generate per-host TLS certificates on the fly. A few details matter here:

- **`basicConstraints=critical,CA:TRUE`** marks the certificate as a CA. Without this, TLS implementations will reject any certificates it signs.
- **`keyUsage=critical,keyCertSign`** grants permission to sign other certificates. iron-proxy cannot issue per-host certs without this.
- **`-days 1`** gives the CA a one-day lifetime. It only needs to survive a single CI run, so keeping it short limits exposure.
- **`-nodes`** leaves the private key unencrypted so iron-proxy can read it without a passphrase.

The CA is ephemeral: created fresh on every run and discarded when the runner is torn down.

### Trust the CA

```hosted-actions-example/.github/workflows/ci.yaml#L39-44
# Trust the CA system-wide, and within Node.js. Some tools require extra config
echo "=== Trusting CA system-wide ==="
sudo cp /tmp/iron-proxy-ca/ca.crt \
  /usr/local/share/ca-certificates/iron-proxy-ca.crt
sudo update-ca-certificates
echo "NODE_EXTRA_CA_CERTS=/tmp/iron-proxy-ca/ca.crt" >> $GITHUB_ENV
```

For TLS interception to work transparently, tools in your CI job need to trust the CA:

- **System trust store:** `update-ca-certificates` adds the CA to the system bundle. This covers tools that use OpenSSL or the system certificate store, including `curl`, `wget`, and `apt`.
- **Node.js:** Node.js ships its own certificate bundle and ignores the system store. Setting `NODE_EXTRA_CA_CERTS` tells it to trust the CA as well. Writing it to `$GITHUB_ENV` makes it available in all subsequent workflow steps.

> **Note:** Other runtimes may need their own configuration. For example, Python's `requests` library respects `REQUESTS_CA_BUNDLE`, and Java uses a keystore that can be updated with `keytool`.

### Stop systemd-resolved

```hosted-actions-example/.github/workflows/ci.yaml#L46-48
          # Stop systemd-resolved. Required to allow iron-proxy to handle DNS resolution
          echo "=== Stopping systemd-resolved ==="
          sudo systemctl stop systemd-resolved || true
```

On Ubuntu runners, `systemd-resolved` manages DNS and listens on port 53. iron-proxy needs that port for its own DNS server. Stopping `systemd-resolved` frees it up.

### Start iron-proxy with setsid

```hosted-actions-example/.github/workflows/ci.yaml#L50-55
# Start iron-proxy. Use setsid to run it in a new session so it is not
# killed when the run: block's shell exits.
echo "=== Starting iron-proxy ==="
sudo install -m 644 -o root -g root /dev/null /var/log/iron-proxy.log
sudo setsid bash -c '/usr/local/bin/iron-proxy -config ./iron-proxy.yaml &>/var/log/iron-proxy.log' &
sleep 0.5
```

iron-proxy runs as root so it can bind to privileged ports (53, 80, 443) without extra configuration.

**`setsid`** starts iron-proxy in a new session, fully detached from the shell's process group. This is critical: GitHub Actions sends signals (like `SIGHUP`) to the shell's process group when a `run:` block ends. Without `setsid`, iron-proxy would be killed between steps. `setsid` moves it into its own session so it survives for the entire workflow.

### Delete the CA Key from Disk

```hosted-actions-example/.github/workflows/ci.yaml#L57-58
# Delete the key from disk now that it's in memory
rm /tmp/iron-proxy-ca/ca.key
```

Once iron-proxy has loaded the CA key into memory, the file on disk is no longer needed. Deleting it limits the window during which a compromised dependency or build script could read it. iron-proxy continues to use the key from memory for the rest of the run.

### Route DNS Through the Proxy

```hosted-actions-example/.github/workflows/ci.yaml#L60-62
# Route DNS through the proxy
echo "=== Routing DNS through the proxy ==="
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolv.conf'
```

This overwrites `/etc/resolv.conf` so all DNS queries go to `127.0.0.1`, where iron-proxy is listening. From this point forward, every hostname resolution goes through the proxy.

### Lock Down Outbound Traffic with iptables

```hosted-actions-example/.github/workflows/ci.yaml#L64-73
# Lock down outbound traffic with iptables. Only loopback traffic (how
# processes reach the proxy), DNS to the upstream resolver, and traffic
# from root (how the proxy reaches the internet) are allowed. Everything
# else is rejected.
echo "=== Configuring iptables ==="
sudo iptables -A OUTPUT -o lo -j ACCEPT
sudo iptables -A OUTPUT -m owner --uid-owner root -j ACCEPT
sudo iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -j REJECT --reject-with icmp-port-unreachable
```

This is the final layer of enforcement. DNS redirection alone can be bypassed if a process connects to a hardcoded IP address or uses its own DNS resolver. The iptables rules on the `OUTPUT` chain close that gap:

1. **`-o lo -j ACCEPT`** allows all traffic on the loopback interface. This is how every process on the runner reaches the proxy (since DNS resolves all hosts to `127.0.0.1`).
2. **`-m owner --uid-owner root -j ACCEPT`** allows root to make outbound connections. iron-proxy runs as root, so this is what lets the proxy reach the real internet on behalf of proxied clients.
3. **`-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`** allows packets on connections that were already open before the rules were applied. This keeps the GitHub Actions runner's pre-existing control connection to the Actions service alive. Without this, the runner could lose contact with GitHub and the job would hang.
4. **`-j REJECT --reject-with icmp-port-unreachable`** rejects everything else with an immediate error.

With these rules in place, a process that tries to `curl` a raw IP address (bypassing DNS entirely) will get an immediate connection error instead of reaching the internet.

## License

This example is provided under the [MIT License](LICENSE).
