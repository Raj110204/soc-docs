# Live Demo

A complete end-to-end walkthrough of the Self-Calibrating SOC Platform in action — from attack simulation to model retraining.

---

## Step 1: Attack Simulation

An SSH brute-force attack is simulated from Kali Linux sending 20 failed login attempts to vm-collection (192.168.100.10). This triggers Wazuh Rule 5710 and starts the full pipeline chain.

![Figure 1 - SSH brute-force simulation from Kali Linux](assets/images/report/figure-4-01.png)

*Figure 1 — SSH brute-force simulation command running from Kali Linux — for loop sending failed SSH login attempts to vm-collection*

---

## Step 2: Wazuh Detection

Wazuh Manager on vm-collection immediately detects Rule 5710 (SSH brute-force) and generates alerts in real time. Every failed login attempt from the attacker IP is logged and forwarded to Elasticsearch.

![Figure 2 - Wazuh Manager detecting Rule 5710](assets/images/report/figure-4-02.png)

*Figure 2 — Wazuh Manager on vm-collection detecting Rule 5710 — live log output showing repeated failed login attempts from 192.168.100.50*

---

## Step 3: Kibana Indexing

Alerts flow from Wazuh into Elasticsearch and are immediately visible in Kibana Discover under the `soc-alerts-*` index. The raw alert documents are now ready for the AI pipeline to process.

![Figure 3 - Kibana Discover soc-alerts index](assets/images/report/figure-4-03.png)

*Figure 3 — Kibana Discover showing soc-alerts index with Wazuh brute-force alert documents, alert_source, alert_type, and field statistics panel*

---

## Step 4: AI Pipeline Scoring

The Python AI pipeline on vm-ai picks up the unscored alerts, maps them to MITRE ATT&CK technique T1110, applies the asset criticality tier multiplier, and assigns a risk_score of 90.0 — flagging this as a high-priority incident.

![Figure 4 - SOC AI pipeline logs, risk_score=90, T1110](assets/images/report/figure-4-04.png)

*Figure 4 — SOC AI pipeline logs confirming alerts found, risk_score assigned (90.0), and MITRE technique T1110 tagged — pipeline processing in real time*

---

## Step 5: TheHive Case Auto-Created

Because the risk score crosses the threshold, the pipeline automatically creates cases in TheHive 5. No analyst intervention required — 14 cases were created and tagged with rule IDs and severity labels.

![Figure 5 - TheHive Cases dashboard auto-created](assets/images/report/figure-4-05.png)

*Figure 5 — TheHive Cases dashboard showing auto-created cases from the AI pipeline with SSH brute-force incidents, severity tags, and linked alerts*

---

## Step 6: Analyst Feedback Written to PostgreSQL

The analyst reviews cases in TheHive and closes them as TruePositive or FalsePositive. The feedback poller on vm-ai picks up these decisions and writes them to the `analyst_decisions` table in PostgreSQL — closing the learning loop.

![Figure 20 - feedback_poller.py writing decisions to PostgreSQL](assets/images/report/figure-4-20.png)

*Figure 20 — vm-ai terminal running feedback_poller.py manually; output showing cases found and analyst decisions written to PostgreSQL analyst_decisions table*

---

## Step 7: Decision Milestone Reached

After 51 analyst decisions are stored, the platform has enough data to retrain the XGBoost model on real environment behaviour. This milestone is the trigger point for calibration.

![Figure 29 - PostgreSQL COUNT(*) = 51 decisions](assets/images/report/figure-4-29.png)

*Figure 29 — PostgreSQL SELECT COUNT(*) FROM analyst_decisions returning 51 — milestone reached to enable XGBoost retraining*

---

## Step 8: XGBoost Model Retraining

`retrain.py` loads the 51 decisions, trains a new XGBoost model, achieves 0.73 accuracy, and saves it to `models/xgb_model.pkl`. The pipeline automatically loads the new model on next restart — the platform has self-calibrated.

![Figure 30 - retrain.py output, accuracy 0.73, model saved](assets/images/report/figure-4-30.png)

*Figure 30 — vm-ai terminal retrain.py output: loaded decisions, class balance shown, XGBoost training complete, accuracy 0.73, model saved to models/xgb_model.pkl*

---

## Step 9: Auto-Isolate Activated

For score >= 90 alerts, the pipeline triggers the Shuffle SOAR Auto-Isolate workflow automatically. The 4-node chain fires: webhook → Wazuh active response (block IP) → TheHive case creation → Slack notification.

![Figure 47 - Auto-isolate webhook live in pipeline](assets/images/report/figure-4-47.png)

*Figure 47 — vm-ai terminal main_pipeline.py updated with auto-isolate webhook call at score >= 90, pipeline restarted and confirmed active with auto-isolate logic live*

---

## Step 10: Live SOC Dashboard

The final Kibana SOC Dashboard gives real-time visibility across the entire platform — alert volume, risk score distribution, top alert types, and alerts by asset tier — all populated with live data.

![Figure 48 - Kibana SOC Dashboard 4 panels live](assets/images/report/figure-4-48.png)

*Figure 48 — Kibana SOC Dashboard showing Alert Volume Over Time, Risk Score Distribution, Top Alert Types, and Alerts by Asset Tier panels with real data*

---

## Platform Results

| Metric | Baseline | Platform Result | Improvement |
|---|---:|---:|---:|
| Alert triage time per shift | 4.5 hours | 1.2 hours | 73% reduction |
| False positive rate | 68% | 22% | 46 percentage point reduction |
| Alert volume reaching analyst | 100% | 1.8% | 98.2% reduction |
| MITRE technique coverage | 0% | 83% | Full coverage from zero |
| Model precision | 0.71 | 0.84 | 18% improvement |
| Mean time to case creation | 5–15 minutes | 4.1 seconds | 99% reduction |