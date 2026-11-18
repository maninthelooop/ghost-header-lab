# ghost-header-lab

**CVE-2026-76835** — Authentication bypass via client-controlled `X-Forwarded-Uri` header in oauth2-proxy 7.15.3.

Isolated Docker lab for SOC and Red Team training. The Red Team emulates adversary behavior exploiting the CVE; Suricata IDS captures the full attack chain in real time and feeds a Grafana SOC dashboard via Loki.

---

## The Vulnerability

oauth2-proxy 7.15.3 configured with `reverse_proxy = true` evaluates `skip_auth_routes` against the `X-Forwarded-Uri` header when present. This header is intended to be set internally by the upstream proxy — not by the client. With no `trusted_proxy_ip` restriction, a client can inject `X-Forwarded-Uri: /public` on any request, causing oauth2-proxy to skip authentication entirely and forward the request to the backend.

**Affected condition:** `reverse_proxy = true` + `skip_auth_routes` defined + no `trusted_proxy_ip`.

---

## Architecture

```
attacker
   │
   ▼
nginx (reverse proxy)
   │  sets X-Forwarded-Uri internally on legitimate requests
   ▼
oauth2-proxy 7.15.3  ← vulnerable: trusts client-supplied X-Forwarded-Uri
   │
   ▼
backend (Flask)
   │  /admin  →  protected
   │  /flag   →  protected
   └  /public →  skip_auth_route (legitimate bypass)

Suricata IDS (inline, monitors all HTTP traffic)
   └→ eve.json → Promtail → Loki → Grafana SOC dashboard
```

---

## Prerequisites

- Docker Engine 24+
- Docker Compose v2
- IP forwarding enabled on the host:

```bash
sysctl -w net.ipv4.ip_forward=1
```

To persist across reboots:

```bash
echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
```

---

## Quickstart

```bash
git clone https://github.com/maninthelooop/ghost-header-lab.git
cd ghost-header-lab/cve-lab
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
docker compose up -d
```

Wait ~15 seconds for all services to initialize, then:

| Service | URL |
|---------|-----|
| Target application | `http://localhost` |
| Grafana SOC dashboard | `http://localhost:3000` (Credentials -> soc:soc-lab-2026) |

---

## Lab Network

The lab uses an isolated internal Docker network. All containers communicate over `172.21.0.0/24`. Suricata monitors the `eth0` interface of the network namespace — no traffic leaves the host.

---

## Red Team — Attack Chain

The exploit does not require credentials, prior authentication, or brute force. The full chain is:

**Phase 1 — Reconnaissance**
Query the oauth2-proxy metrics endpoint to identify the version and confirm the technology stack. Access the `/public` route directly to enumerate which routes bypass authentication by design.

**Phase 2 — Surface Analysis**
Review the official oauth2-proxy documentation for `reverse_proxy` and `skip_auth_routes` configuration behavior. Identify that the proxy evaluates `skip_auth_routes` against `X-Forwarded-Uri` when `reverse_proxy = true` and the header originates from the client.

**Phase 3 — Exploitation**
Send an HTTP request to a protected route (`/admin`, `/flag`) with a client-supplied `X-Forwarded-Uri: /public` header. oauth2-proxy matches `/public` against `skip_auth_routes`, skips authentication, and forwards the request to the backend. The backend returns HTTP 200 with the protected resource.

**Phase 4 — Post-Exploitation**
Confirm access to `/flag` via the same technique. Observe that the bypass works regardless of session state, cookies, or tokens — the header alone is sufficient.

> **Note:** This CVE does not require fuzzing or directory enumeration. Reading the documentation is sufficient to understand the vector.

---

## SOC Dashboard

Grafana dashboard available at `http://localhost:3000` → **CVE Lab — SOC Monitor**.

### Alert Categories

| Tile | Color | SIDs | Trigger |
|------|-------|------|---------|
| RECON | Blue | 2026001, 2026002, 2026007 | Metrics endpoint access, /public enumeration, volumetric request rate |
| SUSPECT | Yellow | 2026003 | `X-Forwarded-Uri` header present in inbound client request |
| ATTACK | Red | 2026004, 2026005, 2026006 | Confirmed auth bypass targeting `/admin`, `/flag`, or X-Forwarded-Uri spoofed to `/public` |

### Reading the Kill Chain

```
RECON only              → discovery phase
RECON + SUSPECT         → adversary identified the vector
RECON + SUSPECT + ATTACK → active exploitation confirmed
ATTACK without RECON    → adversary with prior knowledge, going straight to exploit
```

ATTACK alerts with HTTP status `200` in the log confirm the bypass was successful, not just attempted.

### Fuzzing Detection (SID 2026007)

Threshold-based detection: 20+ requests from the same source IP within 10 seconds triggers a RECON alert with a direct reference to the oauth2-proxy documentation. This detection is independent of tool, User-Agent, or request content.

---

## Suricata Rules

Rules located at `suricata/rules/lab.rules`. Seven signatures covering the full attack chain from reconnaissance to confirmed exploitation.

To verify rules loaded correctly:

```bash
docker logs cve-lab-suricata-1 2>&1 | grep "rules successfully loaded"
```

Expected output: `7 rules successfully loaded, 0 rules failed`

---

## Lab Reset

Full reset including all logs and Loki data:

```bash
docker compose down -v
docker compose up -d
```

---

## Default Credentials

| Service | Username | Password |
|---------|----------|----------|
| Grafana | admin | admin |
| Lab application (Dex) | admin@lab.internal | *(intentionally omitted — bypass the auth, don't brute force it)* |

---

## License

[MIT](../LICENSE)
