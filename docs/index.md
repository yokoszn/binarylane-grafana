# BinaryLane Grafana — Documentation

This documentation teaches you how to use the BinaryLane API with the
[Grafana Infinity datasource](https://grafana.com/grafana/plugins/yesoreyeram-infinity-datasource/)
to build dashboards for your BinaryLane account — and where the limits are.

## Prerequisites

- A running Grafana instance (self-hosted or Grafana Cloud)
- A BinaryLane account with at least one server
- A BinaryLane API token ([how to generate one](02-setup.md#generating-an-api-token))

## What you can monitor

| Area | Doc | What you get |
|------|-----|--------------|
| How it works | [01-how-it-works.md](01-how-it-works.md) | Architecture, auth, Infinity internals |
| Setup | [02-setup.md](02-setup.md) | Datasource config, connection testing |
| Server metrics | [03-server-metrics.md](03-server-metrics.md) | CPU, network, storage, memory per server |
| Fleet overview | [04-fleet-overview.md](04-fleet-overview.md) | All servers, inventory, transfer pooling |
| Billing | [05-billing.md](05-billing.md) | Balance, charges, invoices, account status |
| Alerts & Actions | [06-alerts-actions.md](06-alerts-actions.md) | Threshold alerts, action audit log |
| Variables & Filters | [07-variables-filters.md](07-variables-filters.md) | Dropdowns, filters, transformations |
| Limitations | [08-limitations.md](08-limitations.md) | Full constraints reference |

## Where to start

- **New to this setup?** → [01-how-it-works.md](01-how-it-works.md)
- **Ready to configure the datasource?** → [02-setup.md](02-setup.md)
- **Want to monitor a specific server?** → [03-server-metrics.md](03-server-metrics.md)
- **Want an account-wide view?** → [04-fleet-overview.md](04-fleet-overview.md)
- **Want to track spend?** → [05-billing.md](05-billing.md)
- **Want to track alerts or audit changes?** → [06-alerts-actions.md](06-alerts-actions.md)
- **Something not working?** → [08-limitations.md](08-limitations.md)
