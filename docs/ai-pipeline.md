# AI Pipeline

The `vm-ai` pipeline is the project's main differentiator. It converts raw SIEM alerts into prioritized, contextual incidents that improve as analysts close cases.

## Processing Stages

| Stage | Implementation | Output |
| --- | --- | --- |
| Polling | Python Elasticsearch client | New unprocessed alerts |
| Deduplication | Redis fingerprint cache | `alert_count` and unique incident identity |
| MITRE mapping | `RULE_TO_MITRE` dictionary | Technique IDs and correlation context |
| Asset criticality | JSON asset graph | Tier multiplier for target host |
| Risk scoring | XGBoost model | 0-100 `risk_score` |
| LLM guidance | Claude API | What happened, why it matters, what to check next |
| Case workflow | Shuffle and TheHive APIs | Case, tags, observables, and response tasks |
| Feedback | TheHive poller and PostgreSQL | Analyst decisions for retraining |

## Risk Score Inputs

- `alert_count`: repeated events collapsed into one incident.
- `asset_tier`: target host business criticality.
- `technique_weight`: impact weighting from MITRE ATT&CK mapping.
- `alert_source`: Wazuh, Suricata, Sysmon, or Linux telemetry.
- `false_positive_history`: previous dismissals for a similar fingerprint.

## Thresholds

| Score | Severity | Automation |
| ---: | --- | --- |
| 90-100 | Critical | TheHive case plus Shuffle auto-isolation |
| 70-89 | High | TheHive case with LLM guidance |
| 50-69 | Medium | Review in Kibana and monitor for pattern growth |
| 0-49 | Low | Track as background signal unless volume spikes |

## Feedback Learning

Analyst decisions are stored in PostgreSQL by the feedback poller. The retraining job reads these decisions weekly and replaces the production XGBoost model when a newer model performs better.

The first retraining cycle processed 50+ decisions and improved precision from `0.71` to `0.84`. The most important features after retraining were `asset_tier`, `alert_count`, and `technique_weight`.
