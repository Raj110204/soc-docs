# SOAR & Response

The response layer uses Shuffle SOAR and TheHive 5 to turn high-risk incidents into analyst-ready cases and automated containment actions.

## Playbook 1 - TheHive Case Creation

**Trigger:** AI pipeline scores an incident `>= 70`.

**Actions:**

- Create a TheHive case with title, severity, source, affected host, and risk score.
- Add MITRE ATT&CK technique tags.
- Include the LLM investigation brief in the case description.
- Attach observables such as source IP, target host, rule ID, and alert fingerprint.

## Playbook 2 - Auto-Isolate on Critical Score

**Trigger:** AI pipeline scores an incident `>= 90`.

**Actions:**

- Call Wazuh active response to block the source IP on the affected host.
- Create or update the TheHive case with `CRITICAL-AUTO-ISOLATED`.
- Notify the analyst through Shuffle's configured channel.

## Analyst Decision Loop

| TheHive Resolution | Stored Action | Model Effect |
| --- | --- | --- |
| TruePositive / Resolved | `accepted` | Score remains stable or increases for similar alerts |
| FalsePositive | `dismissed` | Score decreases for matching fingerprints after retrain |
| Duplicate | `dismissed` | Treated as duplicate/noise for future scoring |

## False Positive Genealogy

Every dismissed alert stores a reason, source, analyst, and timestamp. When a similar alert appears later, the pipeline can surface the previous dismissal history inside the case description so analysts do not repeat the same investigation.

## Shuffle Workflow — Auto-Isolate on Critical Score

The Auto-Isolate workflow is a 4-node chain in Shuffle SOAR triggered by the AI pipeline webhook when `risk_score >= 90`.

![Figure 46 - Shuffle Auto-Isolate workflow 4-node chain](assets/images/report/figure-4-46.png)

*Figure 46 — Shuffle SOAR workflow editor showing the Auto-Isolate playbook: High Risk Alert webhook → Wazuh active response block IP → TheHive Create Case → Slack Notify*

| Node | Action | Detail |
|---|---|---|
| 1 — Webhook trigger | Receives POST from pipeline | Payload includes `risk_score`, `source_ip`, `incident_id` |
| 2 — Wazuh active response | Blocks source IP | Calls Wazuh API `active-response` endpoint on the affected host |
| 3 — TheHive case | Creates or updates case | Tags case as `CRITICAL-AUTO-ISOLATED` with full incident context |
| 4 — Slack notify | Alerts analyst channel | Sends risk score, source IP, and TheHive case link |

## Webhook Reference

The pipeline sends a POST to the Shuffle webhook on every scored incident above the critical threshold:

```python
payload = {
    "incident_id": incident_id,
    "risk_score": risk_score,
    "source_ip": source_ip,
    "target_host": target_host,
    "mitre_techniques": mitre_list,
    "alert_count": alert_count
}
requests.post(SHUFFLE_WEBHOOK_URL, json=payload, timeout=10)
```

Webhook URL is stored in `SHUFFLE_WEBHOOK_URL` inside `/opt/soc-ai/.env` on `vm-ai`.
