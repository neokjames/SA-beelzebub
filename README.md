# SA-beelzebub — Beelzebub Honeypot Dashboards and Alerts

Splunk Supporting Add-on providing dashboards and saved searches for the Beelzebub honeypot. Requires [TA-beelzebub](https://github.com/neokjames/TA-beelzebub) to be deployed on all indexers and search heads.

## What's included

| File | Purpose |
|------|---------|
| `default/macros.conf` | Index macro + novelty retention macro |
| `default/savedsearches.conf` | Pre-built reports, alerting rules, novelty trackers, and KV maintenance |
| `default/collections.conf` | KV store collections that persist first-seen state per entity |
| `default/transforms.conf` | KV store lookup definitions exposing each collection |
| `default/data/ui/views/beelzebub_overview.xml` | Overview dashboard |
| `default/data/ui/views/beelzebub_novelty.xml` | Novel activity tracker dashboard |

## Configuration

### Index name

All searches use the `beelzebub_index` macro defined in `default/macros.conf`. The default is `index=honeypot`. If your Beelzebub events land in a different index, edit the macro definition before deploying:

```ini
[beelzebub_index]
definition = index=<your-index-name>
```

## Dashboards

### Beelzebub Honeypot Overview

General-purpose operational view. Filterable by protocol and time range.

| Panel | Description |
|-------|-------------|
| KPIs | Total interactions, unique attacker IPs, interactive sessions, LLM rate limit events, unique credentials |
| Attack Volume Over Time | Stacked area chart by protocol (1h buckets) |
| Top 15 Attacker IPs | Bar chart by event count |
| Protocol Breakdown | Pie chart |
| SSH Client Fingerprints | Scanner and tool identification by client string |
| Top Username / Password Pairs | Most common credential combinations |
| Credential Spray | IPs attempting more than one distinct password |
| Top Targeted HTTP URIs | Most scanned paths and methods |
| HTTP User Agents | Scanner and browser identification |
| LLM Rate Limit — Throttled Attackers | IPs hitting the rate limit with command counts |
| LLM Rate Limit — Throttled Commands | Most frequently blocked commands |
| Interactive Session Commands | Commands issued in confirmed attacker sessions |
| Recent Interactions | Latest raw events |

### Beelzebub — Novel Activity Tracker

Surfaces entities appearing for the first time by querying per-entity KV store collections (`beelzebub_first_seen_*`) populated hourly by the `Beelzebub - Tracker - *` saved searches. The time picker filters by the stored `first_seen` epoch.

**The tracker fills organically.** After installation the dashboard is empty until the first hourly tracker run completes (typically within an hour). Subsequent rows accumulate as new entities are observed; existing entities are never re-flagged. To pre-populate from existing history, run each tracker manually once with `dispatch.earliest_time=0`.

| Panel | Source collection |
|-------|-------------------|
| New Sources (src_ip + client) | `beelzebub_first_seen_source` |
| New Credential Pairs | `beelzebub_first_seen_credential` |
| New HTTP URIs | `beelzebub_first_seen_uri` |
| New HTTP User Agents | `beelzebub_first_seen_user_agent` |
| New Interactive Commands | `beelzebub_first_seen_command` |

## Saved Alerts

| Alert | Schedule | Trigger |
|-------|----------|---------|
| Beelzebub - Credential Spray Alert | Every 15 min | Single IP tries > 10 distinct passwords in 1 hour |
| Beelzebub - High Volume Attacker Alert | Every 30 min | Single IP exceeds 100 events in 1 hour |
| Beelzebub - Interactive Session Started Alert | Every 5 min | Any event with `Status=Interaction` |

## Saved Reports

| Report | Description |
|--------|-------------|
| Beelzebub - Top Attacker IPs | Top 20 source IPs by event count (last 24h) |
| Beelzebub - Protocol Breakdown | Event counts per honeypot protocol (last 24h) |
| Beelzebub - Top Credentials | Most common username/password pairs (last 24h) |
| Beelzebub - Commands Issued | Commands run in interactive sessions (last 24h) |

## Scheduled Trackers (KV store population)

Each tracker scans the previous full hour and inserts any previously-unseen entities into its KV store collection. Schedules are staggered to avoid contention.

| Tracker | Schedule | Collection |
|---------|----------|------------|
| Beelzebub - Tracker - First Seen Source | `5 * * * *` | `beelzebub_first_seen_source` |
| Beelzebub - Tracker - First Seen Credential | `7 * * * *` | `beelzebub_first_seen_credential` |
| Beelzebub - Tracker - First Seen Command | `9 * * * *` | `beelzebub_first_seen_command` |
| Beelzebub - Tracker - First Seen URI | `11 * * * *` | `beelzebub_first_seen_uri` |
| Beelzebub - Tracker - First Seen User Agent | `13 * * * *` | `beelzebub_first_seen_user_agent` |

The source tracker keys on `(src_ip, client)` so a known IP switching SSH/TELNET client tool registers as a new source. For non-SSH/TELNET protocols, `client` is empty.

## KV Store Retention

Five `Beelzebub - Maintenance - Prune First Seen *` saved searches run weekly on Sunday between 03:30 and 03:50, each pruning one collection. Rows whose `first_seen` is older than the `beelzebub_novelty_retention` macro (default `-365d`) are dropped via `inputlookup | where | outputlookup`. An `eventstats` guard aborts the write if the filter would empty the collection, so a misconfigured retention macro is a no-op rather than a data wipe.

- **Tune retention**: edit the `beelzebub_novelty_retention` macro definition in `default/macros.conf` (must be a negative offset accepted by `relative_time()`).
- **Manual one-off prune**: run the relevant `Beelzebub - Maintenance - …` search ad-hoc.
- **Full reset of a collection**: `| makeresults | head 0 | outputlookup beelzebub_first_seen_<entity>`. Use with care; trackers will re-populate as new entities arrive.
