# BinaryLane Grafana Monitoring Stack

Grafana dashboards for BinaryLane cloud servers using the [Infinity datasource](https://grafana.com/grafana/plugins/yesoreyeram-infinity-datasource/) to query the BinaryLane REST API directly — no agent, no exporter, no intermediate database required (except for memory metrics, which need the mPanel agent).

---

> ## ⚠️ Proof of Concept — Read Before Deploying
>
> **This project is a proof of concept.** Querying the BinaryLane API directly from
> Grafana has a significant security implication that you must understand before use:
>
> **BinaryLane API tokens are full-account credentials.** There is currently no way to
> issue a read-only or scoped token — any token you generate has complete access to
> your account, including destructive operations (server deletion, rebuilds, etc.).
> A leaked or compromised token is a serious incident, not just a data exposure.
>
> **Before deploying this stack, you must:**
>
> - **Never commit the token to source control.** Use `.env` (already gitignored) or
>   a secrets manager. Verify with `git log --all --full-diff -p | grep -i bl_api_token`
>   before pushing to any remote.
> - **Restrict Grafana access tightly.** Anyone who can open the Grafana UI and inspect
>   panel queries, browser network traffic, or datasource config can recover the token.
>   Enforce authentication — do not use `GF_AUTH_ANONYMOUS_ENABLED=true` in any
>   deployment that holds a real token.
> - **Protect the Grafana host.** The token lives in Grafana's encrypted secrets store
>   on disk. Treat the Grafana host with the same care as any system that holds
>   account credentials. Restrict SSH access and keep the host patched.
> - **Rotate the token if exposure is suspected.** Generate a new token in mPanel
>   immediately and update your deployment. There is no rate-limiting or IP restriction
>   on BinaryLane API tokens.
> - **Do not expose Grafana to the public internet** without authentication, a reverse
>   proxy with access controls, or VPN/allowlist restrictions in front of it.
>
> If any of the above controls are not in place, do not use a real API token.

---

## Dashboards

| File | Title | Refresh | What it shows |
|---|---|---|---|
| `01-server-metrics.json` | Server Metrics | 1m | CPU, memory, network, storage I/O, transfer quota — per server |
| `02-fleet-overview.json` | Fleet Overview | 5m | All servers, total vCPU/RAM/disk, data transfer pool across account |
| `03-billing.json` | Billing & Account | 10m | Balance, unbilled charges, invoice history, failed payment alerts |
| `04-alerts-actions.json` | Alerts & Actions | 30s | Threshold alert states per server, action audit log, in-progress/errored actions |

---

## Quick Start (Docker)

```bash
# 1. Clone / copy this directory to your Grafana host
# 2. Create your .env file
cp .env.example .env
nano .env          # set BL_API_TOKEN

# 3. Start the stack
docker compose up -d

# 4. Open Grafana
open http://localhost:3000
# Login: admin / admin (change this in .env)
```

Everything is provisioned automatically on first boot — datasource and all four dashboards appear under the **BinaryLane** folder.

---

## Manual Install (existing Grafana)

### 1. Install the Infinity plugin

```bash
grafana-cli plugins install yesoreyeram-infinity-datasource
systemctl restart grafana-server
```

Or if running in Docker, add to your `GF_INSTALL_PLUGINS` env var:
```
GF_INSTALL_PLUGINS=yesoreyeram-infinity-datasource
```

### 2. Provision the datasource

```bash
# Edit the file first — replace REPLACE_WITH_YOUR_API_TOKEN or set BL_API_TOKEN env var
cp provisioning/datasources/binarylane.yaml /etc/grafana/provisioning/datasources/
systemctl restart grafana-server
```

Or configure manually in the UI:
- **Connections → Data sources → Add new → Infinity**
- Name: `BinaryLane`
- Authentication: `Bearer Token`
- Token: your API token
- Allowed hosts: `https://api.binarylane.com.au`
- Save & test

### 3. Import dashboards

```bash
cp provisioning/dashboards/binarylane.yaml /etc/grafana/provisioning/dashboards/
cp dashboards/*.json /var/lib/grafana/dashboards/
```

Or import each JSON via **Dashboards → Import → Upload JSON file**.

---

## Generating an API Token

1. Log in to [mPanel](https://home.binarylane.com.au)
2. Click your name (top right) → **API Info**
3. Click **Generate New Token**
4. Copy the token — it won't be shown again

The token has full account access. Treat it like a password.

---

## Memory Metrics Setup

Memory data (`average.memory_usage_bytes`) requires the **mPanel Memory Graph** agent installed on each server. Without it, the memory panels show no data.

### Linux (all distros)

```bash
wget http://mirror.binarylane.com.au/tools/mpanel-memory-graph.tar.gz
sudo tar xfv mpanel-memory-graph.tar.gz -C /
rm mpanel-memory-graph.tar.gz
```

The installer adds a cron job that runs every few minutes and sends a UDP packet on port 21000.

**Debian 12 only** — cron is not in the base image:

```bash
sudo apt install cron
sudo systemctl enable --now cron
```

**Firewall rule** (if you have iptables rules dropping outbound traffic):

```bash
iptables -I OUTPUT -p udp --dport 21000 -j ACCEPT
# Make persistent:
iptables-save > /etc/iptables/rules.v4
```

Allow up to 15 minutes for data to appear after installation.

### Windows

1. Download [mPanelMemoryGraph.msi](http://mirror.binarylane.com.au/tools/mpanelmemorygraph.msi)
2. .NET Framework 2.0+ required
3. Run the installer — creates a Windows service

Firewall rule:
```
netsh advfirewall firewall add rule name="mPanel Memory Graph" action=allow dir=out protocol=UDP remoteport=21000
```

### Verify

Check the **Server Metrics** dashboard memory panel. If it still shows no data after 15 minutes:

```bash
# Check cron is running
systemctl status cron

# Check the script exists
ls -la /usr/local/bin/mpanel-memory-graph /etc/cron.d/mpanel-memory-graph

# Run it manually and check for errors
sudo /usr/local/bin/mpanel-memory-graph

# Check outbound UDP 21000 is not blocked
nc -uz binarylane.com.au 21000 && echo "UDP OK"
```

---

## Choosing the Right Resolution

The `Resolution` variable in server-level dashboards maps to the API's `data_interval` parameter. At `per_page=200`:

| Setting | Points | Covers |
|---|---|---|
| 5 Minutes | 200 | ~16.7 hours |
| 30 Minutes | 200 | ~4.2 days |
| 4 Hours | 200 | ~33 days |
| 1 Day | 200 | ~6.7 months |
| 1 Week | 200 | ~3.8 years |

Match resolution to your time range — if you set the Grafana time range to "last 7 days" but leave resolution at "5 Minutes", the panel will only show the first 16.7 hours of that range (the API returns at most 200 points).

**Suggested combinations:**

| Time range | Resolution |
|---|---|
| Last 1–12 hours | 5 Minutes |
| Last 1–4 days | 30 Minutes |
| Last 2–5 weeks | 4 Hours |
| Last 3–6 months | 1 Day |
| Historical | 1 Week |

---

## Dashboard Details

### 01 — Server Metrics

Single-server deep-dive. Select server from the dropdown at the top.

Panels:
- **CPU Usage %** — average across all vCPUs (100% = all cores fully saturated)
- **Memory Usage** — average bytes (requires agent) + peak MB per period
- **Network Throughput** — inbound and outbound kbps
- **Storage Throughput** — read/write kbps
- **Storage IOPS** — read/write requests per second (high IOPS alongside memory exhaustion = swap thrashing)
- **Transfer Usage %** — gauge showing this server's used vs quota for the billing period
- **Transfer Used/Quota** — raw GB figures
- **All Servers** — quick reference table

### 02 — Fleet Overview

Account-wide summary. No server variable needed.

Panels:
- **Summary stats** — total/active/powered-off server count, total vCPU/RAM/disk
- **Server Inventory** — full table with OS, IPs, host name, host uptime, host status page (non-null = under maintenance)
- **Data Transfer by Server** — all servers' transfer usage for the current period, with column totals

### 03 — Billing & Account

Panels:
- **Account Status** — active/incomplete/warning/locked (warning/locked = contact support urgently)
- **2FA status** — red if disabled (security reminder)
- **Available Credit** — credit balance in AU$
- **Unbilled Total** — charges accrued so far this period, turns yellow at $50, red at $200 (adjust thresholds to suit)
- **Failed Invoices** — count of unpaid invoices with failed payment attempts; red if > 0 (these block new services)
- **Unbilled Charges Breakdown** — itemised table of all current charges; `Ongoing` = running service, `Pending Invoice` = completed charge awaiting invoicing
- **Invoice History** — full paginated invoice list with amounts, tax, paid status, failure counts, and PDF download links

> Invoice PDF links expire 24 hours after the API responds. Refresh the panel to get a fresh link.

### 04 — Alerts & Actions

Panels:
- **Servers Exceeding Alerts** — count of servers currently breaching at least one threshold; red if > 0
- **Threshold Alerts table** — per-server view of all configured alert types with current measured value vs threshold, and last raised/cleared timestamps
- **In Progress / Errored** — stat panels showing current action state counts
- **Recent Actions** — last 200 account-wide actions with inline progress bar for in-progress actions
- **Server Actions** — same filtered to the selected server

Threshold alert types and what triggers them:

| Type | Threshold unit | Notes |
|---|---|---|
| `cpu` | % | Avg CPU; triggers on sustained load |
| `storage-requests` | IOPS avg | High = likely swap usage |
| `network-incoming` | kbps | Spike = possible DoS |
| `network-outgoing` | kbps | Spike = possible compromise/spam |
| `data-transfer-used` | % of quota | Billing protection |
| `storage-used` | % of disk | Full disk = service failures |
| `memory-used` | % of RAM | >100% = swap in use = poor perf |

---

## Troubleshooting

### "No data" on all panels

1. Check datasource is configured: **Connections → Data sources → BinaryLane → Save & test**
2. Verify the token is correct — test with curl:
   ```bash
   curl -H "Authorization: Bearer YOUR_TOKEN" https://api.binarylane.com.au/v2/account
   ```
3. Check Infinity allowed hosts includes `https://api.binarylane.com.au`

### "No data" on memory panels only

See [Memory Metrics Setup](#memory-metrics-setup) above. The agent must be installed and port 21000 outbound UDP must be open.

### Panels only show partial time range

You've hit the 200-sample cap. Switch to a coarser resolution in the **Resolution** variable.

### Transfer usage shows 0 or 404

The correct endpoint is `/v2/data_usages/{server_id}/current` (underscore, not hyphen). If you're editing a panel URL manually, double-check the path.

### Variable dropdown is empty

The server list query is failing. Check:
- Datasource is working (test connection)
- Your account has at least one server
- The `__text`/`__value` column names are set correctly (Infinity ≥ 2.x required)

### Grafana can't reach the API (Docker)

Add to your `docker-compose.yml` environment:
```yaml
GF_SECURITY_CONTENT_SECURITY_POLICY: "false"
```
And ensure the container has outbound internet access.

---

## File Structure

```
binarylane-grafana/
├── .env.example                            ← copy to .env, set BL_API_TOKEN
├── docker-compose.yml                      ← Grafana + Infinity, fully provisioned
├── provisioning/
│   ├── datasources/
│   │   └── binarylane.yaml                 ← Infinity datasource config
│   └── dashboards/
│       └── binarylane.yaml                 ← dashboard auto-loader config
├── dashboards/
│   ├── 01-server-metrics.json
│   ├── 02-fleet-overview.json
│   ├── 03-billing.json
│   └── 04-alerts-actions.json
└── docs/
    └── api-reference.md                    ← full endpoint/schema/query reference
```

---

## Documentation

Full tutorial documentation is in [`docs/`](docs/):

| Doc | What it covers |
|-----|----------------|
| [How it works](docs/01-how-it-works.md) | Infinity architecture, auth, backend parser |
| [Setup](docs/02-setup.md) | Token, datasource config, connection testing |
| [Server metrics](docs/03-server-metrics.md) | CPU, network, storage, memory, resolution system |
| [Fleet overview](docs/04-fleet-overview.md) | Inventory, transfer pooling, joins |
| [Billing](docs/05-billing.md) | Balance, charges, invoices, account status |
| [Alerts & Actions](docs/06-alerts-actions.md) | Threshold alerts, action audit log |
| [Variables & Filters](docs/07-variables-filters.md) | Dropdowns, filters, transformations |
| [Limitations](docs/08-limitations.md) | Everything that doesn't work and why |

---

## Further Reading

- [BinaryLane API reference](https://api.binarylane.com.au/reference/)
- [Infinity datasource docs](https://grafana.com/docs/plugins/yesoreyeram-infinity-datasource/)
- [mPanel Memory Graph setup](https://support.binarylane.com.au/support/solutions/articles/1000022811-mpanel-memory-graph)
- [BinaryLane data pooling policy](https://www.binarylane.com.au/support/solutions/articles/11000053388)
- [BinaryLane getting started with the API](https://support.binarylane.com.au/support/solutions/articles/11000125203-getting-started-with-binarylane-api)
