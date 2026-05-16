# SA-beelzebub — Beelzebub Honeypot Dashboards and Alerts

Splunk Supporting Add-on providing dashboards and saved searches for the Beelzebub honeypot. Requires [TA-beelzebub](https://github.com/neokjames/TA-beelzebub) to be deployed on all indexers and search heads.

## What's included

| File | Purpose |
|------|---------|
| `default/macros.conf` | Index macro — update here if your index name is not `honeypot` |
| `default/savedsearches.conf` | Pre-built reports and alerting rules |
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

Surfaces entities appearing for the first time in the selected window by comparing against all prior indexed data. Useful for identifying new attacker tooling, credential lists, and attack patterns.

| Panel | Description |
|-------|-------------|
| KPIs | New IPs, new credential pairs, new interaction commands, new throttled commands, new HTTP URIs |
| New Attacker IPs | First-seen IPs with protocol and event count |
| New Credential Pairs | First-seen username/password combinations |
| New SSH Client Fingerprints | First-seen client version strings |
| New HTTP URIs | First-seen request paths |
| New HTTP User Agents | First-seen user agent strings |
| New Interactive Commands | First-seen commands where LLM responded |
| New Throttled Commands | First-seen commands that hit the LLM rate limit |

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
