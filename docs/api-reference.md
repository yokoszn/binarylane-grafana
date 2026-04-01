# BinaryLane API — Grafana Infinity Reference

Quick reference for querying the BinaryLane API through the Grafana Infinity datasource.
All endpoints require Bearer token auth — configured once at the datasource level.

---

## Base URL

```
https://api.binarylane.com.au
```

---

## Authentication

All requests use `Authorization: Bearer <token>`.
Configured in the Infinity datasource provisioning — individual panel queries do not need auth headers.

Generate a token: **mPanel → top-right menu → API Info**

---

## Common Infinity Panel Config

### Time series panel (samplesets)

```
Type:          JSON
Source:        URL
Format:        timeseries
URL:           https://api.binarylane.com.au/v2/samplesets/{server_id}?data_interval={interval}&start=${__from:date:iso}&end=${__to:date:iso}&per_page=200
Root selector: sample_sets
Parser:        backend
```

Columns:

| Selector | Text | Type |
|---|---|---|
| `period.end` | Time | timestamp |
| `average.cpu_usage_percent` | CPU % | number |
| `average.memory_usage_bytes` | Memory | number |
| `average.network_incoming_kbps` | Net In | number |
| `average.network_outgoing_kbps` | Net Out | number |
| `average.storage_read_kbps` | Disk Read | number |
| `average.storage_write_kbps` | Disk Write | number |
| `average.storage_read_requests_per_second` | Read IOPS | number |
| `average.storage_write_requests_per_second` | Write IOPS | number |
| `maximum_memory_megabytes` | Peak Mem MB | number |
| `maximum_storage_gigabytes` | Peak Disk GB | number |

### Non-array root objects (balance, account, data_usage)

For endpoints that return a single object rather than an array, set:
```
Root selector:  <object_key>   e.g. "balance" or "account" or "data_usage"
JSON options → root_is_not_array: true
```

---

## Endpoint Reference

### Account

| Endpoint | Method | Returns | Root selector |
|---|---|---|---|
| `/v2/account` | GET | Account object | `account` |
| `/v2/customers/my/balance` | GET | Balance + charges[] | `balance` |
| `/v2/customers/my/invoices` | GET | Paginated invoices | `invoices` |
| `/v2/customers/my/invoices/{id}` | GET | Single invoice | `invoice` |
| `/v2/customers/my/unpaid-payment-failed-invoices` | GET | Failed invoices | `invoices` |

**Account fields:** `email`, `email_verified`, `two_factor_authentication_enabled`, `status` (incomplete/active/warning/locked), `additional_ipv4_limit`, `configured_payment_methods[]`

**Balance fields:** `unbilled_total` (AU$), `available_credit` (AU$), `generated_at`, `charges[]`

**Charge fields:** `description`, `total` (AU$), `ongoing` (bool — true=running service, false=pending invoice), `created`

**Invoice fields:** `invoice_id`, `invoice_number`, `reference`, `amount` (AU$), `tax`, `tax_code`, `paid`, `refunded`, `payment_failure_count`, `created`, `date_due`, `date_overdue`, `invoice_items[]`, `invoice_download_url` (expires 24h), `invoice_view_url` (expires 24h)

---

### Servers

| Endpoint | Method | Returns | Root selector |
|---|---|---|---|
| `/v2/servers` | GET | Paginated servers | `servers` |
| `/v2/servers/{id}` | GET | Single server | `server` |
| `/v2/servers/{id}/threshold_alerts` | GET | Alert config + current values | `threshold_alerts` |
| `/v2/servers/threshold_alerts` | GET | IDs of servers exceeding alerts | `server_ids` |
| `/v2/servers/{id}/actions` | GET | Server action history | `actions` |

**Server fields:** `id`, `name`, `status` (new/active/archive/off), `vcpus`, `memory` (MB), `disk` (GB), `region.slug`, `image.name`, `networks.v4.0.ip_address` (use dot notation — bracket notation unsupported in Infinity backend parser), `host.display_name`, `host.uptime_ms`, `host.status_page`, `created_at`, `cancelled_at`, `failover_ips[]`, `size_slug`, `vpc_id`, `permalink`

**ThresholdAlert fields:** `alert_type`, `enabled`, `value` (threshold), `current_value`, `last_raised`, `last_cleared`

**ThresholdAlertType values:**

| Value | Measures | Unit |
|---|---|---|
| `cpu` | Avg CPU % | % (0–100) |
| `storage-requests` | Avg combined IOPS | requests/s |
| `network-incoming` | Inbound bandwidth | kbps |
| `network-outgoing` | Outbound bandwidth | kbps |
| `data-transfer-used` | Monthly transfer used | % of quota |
| `storage-used` | Disk used | % of total |
| `memory-used` | Virtual memory used | % of RAM (can exceed 100% = swap in use) |

---

### Sample Sets (Performance Metrics)

| Endpoint | Method | Returns | Root selector |
|---|---|---|---|
| `/v2/samplesets/{server_id}` | GET | Paginated sample sets | `sample_sets` |
| `/v2/samplesets/{server_id}/latest` | GET | Most recent sample set | `sample_set` |

**Query parameters:**

| Param | Values | Notes |
|---|---|---|
| `data_interval` | `five-minute`, `half-hour`, `four-hour`, `day`, `week`, `month` | Resolution of data points |
| `start` | ISO8601 datetime | Use `${__from:date:iso}` in Grafana |
| `end` | ISO8601 datetime | Use `${__to:date:iso}` in Grafana |
| `page` | int (default 1) | Pagination |
| `per_page` | 1–200 (default 20) | Always set to 200 |

**Capacity planning — samples per page at 200:**

| Resolution | 200 samples covers |
|---|---|
| `five-minute` | ~16.7 hours |
| `half-hour` | ~4.2 days |
| `four-hour` | ~33 days |
| `day` | ~6.7 months |
| `week` | ~3.8 years |

⚠️ **Memory data requires the mPanel Memory Graph agent** installed on the server.
Agent sends UDP on port 21000. See: https://support.binarylane.com.au/support/solutions/articles/1000022811
On Debian 12: `sudo apt install cron && systemctl enable --now cron` (cron not in base image).

---

### Data Usage (Transfer)

| Endpoint | Method | Returns | Root selector |
|---|---|---|---|
| `/v2/data_usages/{server_id}/current` | GET | Current period usage | `data_usage` |
| `/v2/data_usages/current` | GET | All servers current usage | `data_usages` |

**DataUsage fields:** `server_id`, `current_transfer_usage_gigabytes`, `transfer_gigabytes` (quota), `transfer_period_end`, `expires`

Note: BinaryLane pools data transfer across servers. A server that uses less than its quota offsets one that exceeds it.

---

### Actions (Audit Log)

| Endpoint | Method | Returns | Root selector |
|---|---|---|---|
| `/v2/actions` | GET | Paginated account-wide actions | `actions` |
| `/v2/actions/{id}` | GET | Single action | `action` |
| `/v2/servers/{id}/actions` | GET | Per-server actions | `actions` |

**Action fields:** `id`, `type`, `status` (in-progress/completed/errored), `title`, `reason`, `resource_type`, `resource_id`, `started_at`, `completed_at`, `result_data`, `progress.percent_complete`, `progress.current_step`, `progress.current_step_detail`, `progress.completed_steps[]`

---

### Load Balancers

| Endpoint | Method | Returns | Root selector |
|---|---|---|---|
| `/v2/load_balancers` | GET | Paginated LBs | `load_balancers` |
| `/v2/load_balancers/{id}` | GET | Single LB | `load_balancer` |

Note: LBs have no performance metrics endpoint in the API (no equivalent of samplesets). Only configuration data is available.

---

## Grafana Variable Setup

### Server dropdown (name → ID)

```json
{
  "type": "query",
  "name": "server_id",
  "datasource": "BinaryLane",
  "query": {
    "type": "json",
    "source": "url",
    "url": "https://api.binarylane.com.au/v2/servers?per_page=200",
    "root_selector": "servers",
    "columns": [
      { "selector": "name", "text": "__text", "type": "string" },
      { "selector": "id",   "text": "__value", "type": "string" }
    ]
  }
}
```

`__text` and `__value` are special column names that Infinity uses for variable label/value mapping.

### Resolution dropdown

```
Custom variable: five-minute : 5 Minutes, half-hour : 30 Minutes, four-hour : 4 Hours, day : 1 Day, week : 1 Week
```

---

## Infinity Filter Syntax

Infinity's `backend` parser supports client-side filters on JSON responses:

```json
"filters": [
  { "field": "status", "operator": "equals", "value": "active" },
  { "field": "status", "operator": "equals", "value": "in-progress" }
]
```

Supported operators: `equals`, `not_equals`, `contains`, `not_contains`, `starts_with`, `ends_with`

Filters are OR'd within a single target. For AND logic, use multiple targets + Grafana transformations.

---

## Common Grafana Transformations

### Calculate transfer used %

```
Transform: Calculate field
Mode:      Binary
Left:      Used GB
Operator:  /
Right:     Quota GB
Alias:     Transfer %
```

Then multiply by 100 with a second calculateField transform for display in gauge panels.

### Join server names to IDs (cross-dataset)

Use **Join by field** transformation to merge the servers table (has name+id) with any table that contains a `server_id` column.

---

## Known Limitations

- **No LB health metrics** — load balancer configuration only, no traffic/health data
- **No real-time data** — minimum `five-minute` resolution; dashboard refresh < 5m has no effect on data freshness
- **Memory requires agent** — `average.memory_usage_bytes` will be null/0 without the mPanel agent
- **200 per-page cap** — cannot retrieve more than 200 results per query; paginate manually for large datasets
- **Invoice PDF URLs expire** — `invoice_download_url` and `invoice_view_url` expire 24 hours after the API call
- **Transfer pooling** — data usage figures reflect BinaryLane's pooling policy, not raw per-server usage
- **`CurrentServerAlertsResponse`** returns only an array of `server_ids` (integers), not alert details — join against per-server threshold_alerts calls to get full details
