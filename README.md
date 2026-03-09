<p align="center">
  <img src="docs/public/favicon.png" alt="Slim logo" width="64" height="64" />
</p>

<h1 align="center">Slim</h1>

<p align="center">
  Simple command to get clean HTTPS local domains for your projects
</p>

<p align="center">
  <a href="https://slim.sh"><img src="https://img.shields.io/badge/website-slim.sh-0f172a?style=flat-square" alt="Website"></a>
  <img src="https://img.shields.io/badge/go-1.25%2B-00ADD8?style=flat-square&logo=go&logoColor=white" alt="Go 1.25+">
  <img src="https://img.shields.io/badge/platform-macOS%20%7C%20Linux-111827?style=flat-square" alt="Platform">
</p>

```
myapp.test        → localhost:3000
myapp.test/api    → localhost:8080
dashboard.test    → localhost:5173
app.loc           → localhost:4000
```

## Install

```bash
curl -sL https://slim.sh/install.sh | sh
```

<details>
<summary>Build from source</summary>

```bash
git clone https://github.com/kamranahmedse/slim.git
cd slim
make build
make install
```

Requires Go 1.25 or later.
</details>

## Quick Start

```bash
slim start myapp --port 3000
# https://myapp.test → localhost:3000
```

First run handles all setup automatically (CA generation, keychain trust, port forwarding).

## Usage

### Starting and Stopping

```bash
slim start myapp --port 3000
slim start api --port 8080
slim stop myapp                  # stop one domain
slim stop                        # stop all domains
```

### Custom TLDs

Bare names like `myapp` get `.test` appended automatically. Specify a full domain to use any TLD:

```bash
slim start app.loc --port 3000   # https://app.loc → localhost:3000
slim start my.dev --port 4000    # https://my.dev → localhost:4000
```

> **Note:** Avoid `.local` — it's reserved for mDNS and can cause slow DNS resolution on macOS/Linux.

### Path Routing

Route different URL paths to different upstream ports on a single domain:

```bash
slim start myapp --port 3000 --route /api=8080 --route /ws=9000
```

Routes use longest-prefix matching — `/api/users` matches `/api` before `/`. The `--port` flag sets the default upstream for unmatched paths.

### Project Config (`.slim.yaml`)

Define all services for a project in a `.slim.yaml` file at the project root:

```yaml
services:
  - domain: myapp
    port: 3000
    routes:
      - path: /api
        port: 8080
  - domain: dashboard
    port: 5173
  - domain: app.loc
    port: 4000
log_mode: minimal  # full | minimal | off
cors: true         # enable CORS headers on proxied responses
```

```bash
slim up                              # start all services
slim up --config /path/to/.slim.yaml # specify a config path
slim down                            # stop all project services
```

### Internet Sharing

Expose a local server to the internet with a public `slim.show` URL. Requires `slim login` first.

```bash
slim share --port 3000                              # random subdomain
slim share --port 3000 --subdomain demo             # https://demo.slim.show
slim share --port 3000 --password secret            # password protected
slim share --port 3000 --ttl 30m                    # auto-expires after 30 minutes
slim share --port 3000 --domain myapp.example.com   # custom domain
```

Custom subdomains, custom domains, and password protection require a Pro subscription.

### Logs and Diagnostics

```bash
slim list                # inspect running domains
slim list --json

slim logs                # view access logs
slim logs --follow myapp # tail logs for a domain
slim logs --flush        # clear log file

slim doctor              # run diagnostic checks
```

```
$ slim doctor
  ✓  CA certificate        valid, expires 2035-02-28
  ✓  CA trust              trusted by OS
  ✓  Port forwarding       active (80→10080, 443→10443)
  ✓  Hosts: myapp.test    present in /etc/hosts
  !  Daemon                not running
  ✓  Cert: myapp.test     valid, expires 2027-06-03
```

### Uninstall

```bash
slim uninstall   # removes everything: CA, certs, hosts entries, port-forward rules, config
```

## How It Works

1. **Creates a local CA** — generated on first use and trusted in the system store (macOS Keychain or Linux CA store).
2. **Issues per-domain certificates** — on the fly, signed by the CA and served via SNI.
3. **Updates `/etc/hosts`** — automatically points your domains to `127.0.0.1`.
4. **Forwards ports 80/443** — macOS `pfctl` or Linux `iptables` redirects to unprivileged 10080/10443 so the proxy doesn't need root.
5. **Reverse proxies requests** — Go's `httputil.ReverseProxy` handles HTTP/2 and WebSocket upgrades natively, so HMR for Next.js, Vite, etc. works out of the box.
6. **Tunnels to the internet** — `slim share` creates a WebSocket tunnel to `slim.show`, giving your local server a public HTTPS URL.

Config lives at `~/.slim/config.yaml`. Certificates in `~/.slim/certs/`, root CA in `~/.slim/ca/`, logs in `~/.slim/access.log`.

## License

[PolyForm Shield 1.0.0](./LICENSE) © [Kamran Ahmed](https://x.com/kamrify)
