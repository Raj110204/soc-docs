# Script Reference

All Python scripts in `/home/vboxuser/soc-ai/` on `vm-ai` and their purpose.

---

## `main_pipeline.py`

The core pipeline service. Polls Elasticsearch every 30 seconds, runs deduplication, MITRE mapping, asset scoring, XGBoost risk scoring, LLM guidance, and triggers Shuffle/TheHive.

**Run as service:**

```bash
sudo systemctl start soc-pipeline
sudo systemctl status soc-pipeline
journalctl -u soc-pipeline -f
```

**Run manually (debug):**

```bash
cd /home/vboxuser/soc-ai
python main_pipeline.py
```

**Key parameters:** `POLL_INTERVAL`, `HIGH_RISK_THRESHOLD`, `CRITICAL_THRESHOLD` — see [Configuration Reference](configuration.md).

---

## `mitre_mapper.py`

Maps Wazuh rule IDs and Suricata signature keywords to MITRE ATT&CK technique IDs.

**Function:** `map_alert(alert_doc)` → returns a list of technique strings like `["T1110", "T1059"]`.

**To add a new mapping:**

1. Find the rule ID in Kibana Discover (`rule.id` field).
2. Add the entry to `RULE_TO_MITRE`.
3. Restart the pipeline: `sudo systemctl restart soc-pipeline`.

---

## `asset_graph.py`

Stores host-to-tier mappings and returns a criticality multiplier for a given IP address.

**Function:** `get_asset_tier(ip)` → returns `1`, `2`, or `3`.

Tier 1 hosts receive a `1.5×` score multiplier. All lab VMs and endpoints must be added here for accurate scoring.

---

## `risk_scorer.py`

Builds the feature vector from an enriched alert and runs XGBoost inference.

**Function:** `score_alert(alert)` → returns `risk_score` (0–100).

Features used: `alert_count`, `asset_tier`, `technique_weight`, `alert_source_encoded`, `false_positive_history`.

If the XGBoost model is missing or fails to load, the script falls back to a heuristic scoring formula so the pipeline continues running.

---

## `retrain.py`

Loads analyst decisions from PostgreSQL, trains a new XGBoost model, and saves it to `models/xgb_model.pkl` if performance improves.

**Run manually:**

```bash
cd /home/vboxuser/soc-ai
python retrain.py
sudo systemctl restart soc-pipeline
```

**Minimum decisions required:** 50 (configurable via `MIN_DECISIONS` in the script).

**Output:** accuracy score, class balance, and model save confirmation.

---

## `feedback_poller.py`

Queries TheHive for recently closed cases and writes analyst resolution decisions to the `analyst_decisions` PostgreSQL table.

**Run manually:**

```bash
python feedback_poller.py
```

**Run as cron (recommended):** add to crontab on `vm-ai`:

```bash
# Every hour
0 * * * * /usr/bin/python3 /home/vboxuser/soc-ai/feedback_poller.py >> /var/log/soc-feedback.log 2>&1
```

**PostgreSQL table written:** `analyst_decisions(incident_id, analyst_action, decided_at)`.

---

## `llm_guidance.py`

Calls the local Ollama `phi3` model to generate an investigation brief for each high-risk incident.

**Output format (injected into TheHive case description):**

- What happened
- Why it matters
- What to check next

Timeout is set to 300 seconds. If Ollama is unavailable, the pipeline inserts a fallback message and continues case creation.
