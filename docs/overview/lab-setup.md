# Lab Setup

The lab runs on an isolated internal subnet and is designed for safe testing of SOC workflows, brute-force simulations, endpoint telemetry, SIEM ingestion, and SOAR response.

## Internal Services

| Service | URL | Primary Use |
| --- | --- | --- |
| Kibana | `http://192.168.100.20:5601` | Alert discovery, dashboards, and metric validation |
| Elasticsearch | `http://192.168.100.20:9200` | Internal index/API access for pipeline and Logstash |
| TheHive 5 | `http://192.168.100.40:9000` | Case management and analyst decisions |
| Shuffle SOAR | `http://192.168.100.40:3001` | Playbooks and workflow execution history |

!!! warning "Credential Handling"
    Lab credentials were rotated during Phase 5 hardening. Public documentation should avoid hardcoding passwords; use your vault or private setup notes for operational credentials.

## Baseline Health Checks

| Check | VM | Expected Result |
| --- | --- | --- |
| Wazuh agents active | `vm-collection` | Agents show `Active` |
| Elasticsearch cluster healthy | `vm-siem` | Health is `green` or `yellow` |
| AI pipeline running | `vm-ai` | `soc-pipeline` is `active (running)` |
| TheHive available | `vm-response` | Login page loads |
| Shuffle available | `vm-response` | Dashboard loads |
| Kibana dashboard live | `vm-siem` | SOC dashboard shows recent alerts |

## Test Alert Command

Run a failed-login loop from another host on the lab network to trigger a Wazuh brute-force alert:

```bash
for i in {1..20}; do ssh wronguser@192.168.100.10; done
```

Wazuh rule `5710` should map to MITRE technique `T1110` and produce a scored incident after the pipeline processes the alert.
