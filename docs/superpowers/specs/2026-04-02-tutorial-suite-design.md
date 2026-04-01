# Design: BinaryLane Grafana Tutorial Documentation Suite

**Date:** 2026-04-02
**Status:** Approved
**Audience:** BinaryLane customers with an existing Grafana deployment

---

## Goal

Transform the binarylane-grafana repo from an installation guide + reference into a
task-oriented documentation suite that teaches customers what they can monitor via the
BinaryLane API through the Grafana Infinity datasource — and, critically, what they
cannot, and why.

---

## Audience

BinaryLane customers who already have Grafana running. They are not Grafana beginners.
They arrive with a goal ("I want to track my billing", "I want to see fleet status") and
need to understand both how to achieve it and where the walls are.

---

## Format

Plain markdown files in `docs/`. Renders on GitHub without any tooling. Mermaid diagrams
are rendered natively by GitHub markdown.

---

## Document Structure

```
docs/
  index.md                  ← Navigation, overview of what the suite covers
  01-how-it-works.md        ← Infinity + BL API architecture, auth, proxying
  02-setup.md               ← Datasource config, token generation, testing connection
  03-server-metrics.md      ← Per-server performance panels, samplesets, resolution
  04-fleet-overview.md      ← Fleet inventory, transfer pooling, cross-server joins
  05-billing.md             ← Balance, charges, invoices, account status states
  06-alerts-actions.md      ← Threshold alerts, action audit log, in-progress tracking
  07-variables-filters.md   ← Variables, Infinity filters, Grafana transformations
  08-limitations.md         ← Consolidated limitations reference with doc links
  superpowers/specs/        ← Design docs (this file)
```

---

## Per-Document Shape

Every tutorial document follows this structure:

1. **What you can monitor** — capabilities summary, what data is available
2. **How the data flows** — Mermaid diagram showing API → Infinity → Grafana panel
3. **Building it** — step-by-step Infinity panel config with exact field values
4. **Limitations for this area** — hard limits, workarounds, what is simply not possible

---

## Mermaid Diagrams

One diagram per document. Types and content:

| Document | Diagram type | What it shows |
|---|---|---|
| `01-how-it-works` | Flowchart | Browser → Grafana backend → Infinity → BL API; auth path |
| `02-setup` | Sequence diagram | Token generation → datasource provisioning → first test query |
| `03-server-metrics` | Flowchart | Samplesets endpoint → column mapping → time series panel; resolution coverage |
| `04-fleet-overview` | Entity relationship | servers endpoint → data_usages join; transfer pooling logic |
| `05-billing` | State diagram | Account status states (incomplete/active/warning/locked); charge lifecycle |
| `06-alerts-actions` | State diagram | Alert raised/cleared/disabled; action in-progress/completed/errored |
| `07-variables-filters` | Flowchart | Variable query → URL interpolation → panel data |
| `08-limitations` | Quadrant chart | Capability matrix: works well / partial / needs workaround / not possible |

---

## Content Scope Per Document

### `index.md`
- What this repo is and what it covers
- Prerequisites (existing Grafana, BL account, API token)
- Navigation table linking all docs
- Quick orientation: "if you want X, start at doc Y"

### `01-how-it-works.md`
- What the Infinity datasource does (server-side proxy, not client-side fetch)
- Why auth is configured once at datasource level, not per panel
- How Grafana backend parser vs frontend parser differ and when to use each
- BL API base URL, versioning, pagination model
- Architecture diagram

### `02-setup.md`
- Generating an API token in mPanel
- Configuring the Infinity datasource (UI walkthrough + provisioning YAML)
- Allowed hosts requirement and why it exists
- Testing the connection with a simple query
- Docker Compose path vs manual install path
- Sequence diagram of provisioning flow

### `03-server-metrics.md`
- What the samplesets endpoint provides (CPU, network, storage, memory, disk)
- How to configure a time series panel against samplesets
- The resolution system: `data_interval` parameter values and their coverage at 200 samples
- The `per_page=200` hard cap and what happens when you exceed your time range
- Grafana time range variables (`${__from:date:iso}`, `${__to:date:iso}`)
- Memory data: requires mPanel agent, what happens without it
- Limitations: 5-minute minimum resolution, 200-sample cap, memory agent dependency

### `04-fleet-overview.md`
- The `/v2/servers` endpoint: what fields are available
- Dot notation requirement for nested fields (bracket notation unsupported in backend parser)
- Joining server names to IDs using Grafana's Join by field transformation
- The `/v2/data_usages/current` endpoint and transfer pooling explanation
- Building a fleet inventory table panel
- Limitations: no per-server real-time status, no historical inventory, no LB metrics

### `05-billing.md`
- Account status states and what each means operationally
- Balance endpoint: available credit, unbilled total
- Charges array: ongoing vs pending-invoice distinction
- Invoice list: fields, pagination, PDF URL expiry
- Failed invoice detection and why it matters (blocks new services)
- 2FA status as a security signal
- Limitations: no webhook/alert triggers from Grafana, invoice PDF URLs expire in 24h, no payment method details exposed

### `06-alerts-actions.md`
- Threshold alert types, units, and what each measures
- The two-endpoint alert model: `/v2/servers/threshold_alerts` (IDs only) vs per-server detail
- Why joining these requires multiple Infinity targets + transformations
- Action status lifecycle and what in-progress/errored means
- Building an action audit table with inline progress
- Limitations: no push notifications, no Grafana alerting integration from API data, alert history not available (only current state)

### `07-variables-filters.md`
- Server dropdown variable: `__text`/`__value` pattern
- Resolution dropdown: custom variable with `data_interval` values
- Infinity filter syntax: operators, OR vs AND logic
- Grafana transformations used across the dashboards: Calculate field, Join by field
- How variable interpolation appears in panel URLs

### `08-limitations.md`
- Consolidated reference of every known limitation across all areas
- Each entry: what it is, why it exists, any workaround
- Grouped by area: performance metrics, fleet, billing, alerts/actions, Infinity-specific
- Quadrant chart showing capability landscape

---

## Out of Scope

- Building panels for API endpoints not currently used in the dashboards (VPCs,
  SSH keys, images, regions, sizes) — the API has these but they are config-only
  with no monitoring value
- Grafana alerting configuration (Grafana-native alerts, not BL threshold alerts)
- Any server provisioning, automation, or write operations via the API
- MkDocs/Docusaurus site generation
