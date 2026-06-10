# Zabbix Template for Cato Networks

Monitor your [Cato Networks](https://www.catonetworks.com/) SASE deployment in Zabbix using the Cato GraphQL API. The template polls the API **once** per interval and automatically discovers every active site and socket as its own Zabbix host, then tracks connectivity, uptime and per-WAN/ISP status with sensible trigger dependencies.

- **Zabbix version:** 7.0
- **Data source:** Cato GraphQL API (`accountSnapshot`)
- **License:** MIT (see [LICENSE](LICENSE))

## How it works

```
                 ┌───────────────────────────┐
                 │  Host: "Cato Networks Cloud"│
                 │  Template Cato CMA (Main)   │
                 │  • 1 HTTP request to the API│
                 │  • LLD: sites + sockets     │
                 └──────────────┬──────────────┘
            discovers & creates hosts (read raw data back)
                 ┌──────────────┴──────────────┐
        ┌────────▼─────────┐         ┌──────────▼─────────┐
        │ Cato Site - <x>  │         │ <Type> Cato Socket │
        │ Template Cato Site│        │ Template Cato Socket│
        │ • site up/down   │         │ • socket up/down    │
        │ • per-ISP status │         │ • uptime            │
        └──────────────────┘         │ • per-interface (WAN)│
                                     └──────────────────────┘
```

Only the **Main** template calls the API. The Site and Socket templates are *auto-assigned* by discovery and read the cached snapshot back from the central host via calculated items — so the API is hit just once regardless of how many sites you have.

## Requirements

- Zabbix 7.0 server/proxy with outbound HTTPS to `api.catonetworks.com`.
- A Cato API key with read access (CMA portal → **Resources → Service API Keys**).
- Your Cato **account ID** (visible in the CMA portal URL after _/account/_).

## Installation

1. **Import the template**
   - In Zabbix: **Data collection → Templates → Import** and select `zbx_cato_templates.json`.
   - This creates three templates and the required template/host groups.
   - If import fails, manually create the template/host groups and update the template with the proper group UUIDS.

2. **Create the central host**
   - Create a host named **exactly** `Cato Networks Cloud`.

     > ⚠️ The name must match exactly. The Site and Socket templates reference `last(/Cato Networks Cloud/cato.api.data)` literally; a different host name will silently break all derived metrics.
   - Attach the template **`Template Cato CMA`** to it.
   - Give it any interface (the data comes via HTTP agent, so the interface is not actually used).

3. **Set the macros** on that host (or on the Main template):

   | Macro | Description |
   |-------|-------------|
   | `{$CATO.ACCOUNT.ID}` | Your Cato account ID |
   | `{$CATO.API.KEY}` | Your Cato API key (sent as the `x-api-key` header) |

4. **Wait for discovery.** Within a couple of polling cycles, hosts named `Cato Site - <name>` and `<Primary/Secondary> Cato Socket - <name>` will appear, each with its own metrics and triggers.

## Macros

| Macro | Default | Where | Purpose |
|-------|---------|-------|---------|
| `{$CATO.ACCOUNT.ID}` | `REPLACE_ME` | Main template | Cato account ID used in the GraphQL query |
| `{$CATO.API.KEY}` | `REPLACE_ME` | Main template | API key sent in the `x-api-key` header |

Discovery also sets several per-host macros automatically (`{$SITE_ID}`, `{$SERIAL}`, `{$FULL_SITE_NAME}`, etc.) — you don't need to touch these.

## Triggers

| Template | Trigger | Severity |
|----------|---------|----------|
| Site | Site is disconnected | Disaster |
| Socket | Socket is disconnected | High |
| Socket | Interface `<name>` is down | Average |

### The "Site down (dependency suppressor)" trigger

`Template Cato Socket` contains a trigger named **`Site down (dependency suppressor - do not notify)`** (severity *Information*, tagged `type=dummy`). **This is intentional and is not a bug.**

Zabbix only suppresses a child trigger when its *parent* trigger is in the `PROBLEM` state. When a whole site goes offline, every socket and every interface on it would otherwise alarm at once. The per-socket and per-interface triggers **depend on** this suppressor trigger, so when the entire site is down only this single low-priority event fires and the noisy downstream alerts stay quiet.

**You should exclude it from notifications.** In your alert action conditions, add a condition to *not* notify when the tag `type` equals `dummy`.

## Tags

Discovered hosts are tagged with `class`, `device_type`, `site`, `sn` (serial), and `assignment_group=Zabbix Alarms`. The `assignment_group` value is an example routing convention — change or remove it to match your own alerting/ITSM setup.

## Notes & limitations

- Only sites with `operationalStatus = active` are discovered.
- Per-ISP discovery assumes a dual-socket (HA) site with `WAN1`/`WAN2` interfaces; single-socket sites still report socket and interface status.
- Discovery rules use `DELETE_NEVER` / `DISABLE_NEVER` lifetimes, so removed sites/sockets are **not** auto-removed from Zabbix. Clean them up manually if needed.

## Contributing

Issues and pull requests welcome. Please don't include real account IDs, serials, API keys or site names in examples.
