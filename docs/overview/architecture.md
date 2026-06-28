# Architecture

The project uses a four-VM design on an isolated `192.168.100.0/24` lab network. Each VM owns a clear SOC function so the environment resembles a realistic enterprise security stack.

![Self-Calibrating SOC architecture](../assets/images/report/architecture.png)

## VM Layout

| VM | IP Address | Role | Key Services | RAM | CPU | Disk |
| --- | --- | --- | --- | ---: | ---: | ---: |
| vm-collection | 192.168.100.10 | Log Collection | Wazuh Manager, Suricata IDS, Filebeat | 8 GB | 4 vCPU | 60 GB |
| vm-siem | 192.168.100.20 | SIEM | Elasticsearch, Logstash, Kibana | 16 GB | 8 vCPU | 200 GB |
| vm-ai | 192.168.100.30 | AI Engine | Python Pipeline, XGBoost, PostgreSQL, Redis, Claude API | 8 GB | 4 vCPU | 60 GB |
| vm-response | 192.168.100.40 | Response | TheHive 5, Shuffle SOAR | 16 GB | 6 vCPU | 100 GB |

## End-to-End Flow

1. Wazuh, Suricata, Sysmon, and auditd generate raw telemetry.
2. Filebeat ships normalized logs from `vm-collection` to Logstash on `vm-siem`.
3. Logstash parses fields, normalizes timestamps, tags alert sources, and writes to `soc-alerts-*`.
4. The Python pipeline on `vm-ai` polls Elasticsearch every 30 seconds.
5. Redis stores deduplication fingerprints so repeated alerts update `alert_count` instead of creating duplicate incidents.
6. The MITRE mapper attaches technique IDs such as `T1110`, `T1059`, and `T1071`.
7. The asset graph applies host tier multipliers before XGBoost calculates the final risk score.
8. Scores `>= 70` create TheHive cases with LLM guidance; scores `>= 90` also trigger auto-isolation.
9. The analyst closes the case, and the feedback poller stores the decision in PostgreSQL.
10. Weekly retraining updates the scoring model from analyst decisions.

## Key Design Choices

- **Separation of concerns:** collection, SIEM, AI, and response services are isolated by VM.
- **Open-source first:** every infrastructure component is open-source, with Claude API as the only optional paid dependency.
- **Self-calibration:** analyst decisions are treated as training data, not only case history.
- **Asset-aware scoring:** domain controllers and SOC servers receive higher risk weighting than workstations.
