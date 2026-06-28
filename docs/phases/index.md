# Implementation Phases

The implementation was validated in six phases, moving from a basic end-to-end smoke test to hardened, expanded SOC coverage.

| Phase | Name | Window | Goal |
| --- | --- | --- | --- |
| Phase 1 | Pipeline Smoke Test | Day 1 | Prove that a real alert travels from collection through scoring into TheHive. |
| Phase 2 | Asset Criticality Graph Tuning | Day 1-2 | Make scores reflect business impact by assigning hosts to criticality tiers. |
| Phase 3 | MITRE ATT&CK Coverage Expansion | Week 1 | Map observed Wazuh and Suricata alerts to MITRE techniques for better correlation. |
| Phase 4 | Analyst Feedback Loop | Week 1-2 | Feed TheHive analyst decisions into PostgreSQL so XGBoost can retrain on local reality. |
| Phase 5 | Production Hardening | Week 2 | Replace lab defaults and add baseline security controls before handling real traffic. |
| Phase 6 | Expanded Coverage | Ongoing | Grow the lab into a richer SOC simulation with more endpoints, playbooks, and dashboards. |

## Completion Checklist

| Phase | Done When |
| --- | --- |
| Phase 1 | A real alert is scored and a TheHive case is auto-created |
| Phase 2 | All lab hosts are mapped in `asset_graph.py` with correct tiers |
| Phase 3 | At least 80 percent of observed alerts have MITRE mappings |
| Phase 4 | 50+ analyst decisions exist and retraining has run |
| Phase 5 | Password rotation, Elasticsearch auth, TLS proxying, and snapshots are complete |
| Phase 6 | Multiple endpoints, auto-isolate workflow, and live Kibana dashboard are active |
