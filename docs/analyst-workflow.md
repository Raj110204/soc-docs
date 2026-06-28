# Analyst Workflow

This page condenses the L1 workflow guide into the day-to-day operating process for the platform.

## Shift Start Checklist

- Confirm Wazuh agents are active.
- Check Elasticsearch cluster health.
- Confirm the `soc-pipeline` service is running.
- Open TheHive and note the current open case count.
- Open Kibana SOC Overview and confirm new alert data is visible.
- Check Shuffle workflow runs for failures from the previous shift.

## Triage Flow

1. Open the highest-priority unassigned TheHive case.
2. Assign the case to yourself.
3. Read the LLM case description: what happened, why it matters, and what to check next.
4. Verify source IP, target host, risk score, alert count, asset tier, and MITRE tags in Kibana.
5. Investigate the affected host and surrounding timeline.
6. Add a TheHive observation note with findings and reasoning.
7. Close as TruePositive, FalsePositive, or Duplicate, or escalate to L2.

## Escalate Immediately

- Risk score `>= 90`.
- A Tier 1 host is targeted.
- Shuffle auto-isolation fired.
- Lateral movement, command-and-control, or exfiltration is suspected.
- The analyst cannot classify the incident within 15 minutes.

## Closing False Positives

False positive notes should include the reason, benign source, analyst ID, and date. This structure is important because the model learns from the decision during the next retraining cycle.

```text
FP - {reason: known_scanner, ip: 192.168.100.55, dismissed_by: analyst_01, date: 2026-06-25}
```

## Common Scenarios

### Auto-Isolate Fired But It's a False Positive

When Shuffle auto-isolate triggers on a score `>= 90` and the source turns out to be benign (known scanner, internal tool, or misconfigured host):

1. Open the TheHive case tagged `CRITICAL-AUTO-ISOLATED`.
2. Check the source IP against known internal assets in `asset_graph.py`.
3. If benign — confirm in Kibana that no lateral movement or C2 activity follows the alert.
4. Close the case as **FalsePositive** with a structured note:

```text
FP - {reason: known_internal_scanner, ip: 192.168.100.55, dismissed_by: analyst_01, date: 2026-06-28}
```

5. Notify the team that the IP was blocked by active response — the Wazuh block must be manually removed:

```bash
# On vm-collection
/var/ossec/bin/agent_control -b  -f firewall-drop -u
```

6. Add the IP to an internal allowlist note inside the case for the next retraining cycle.

---

### Tier 1 Host Alert vs Tier 3 Host Alert

The same alert type behaves differently depending on asset tier. Use this as a guide:

| Situation | Tier 1 Host (vm-collection, vm-siem) | Tier 3 Host (WIN-ENDPOINT-01, workstations) |
|---|---|---|
| SSH brute force, score 90 | Escalate immediately, do not wait | Investigate within shift, monitor for follow-on |
| Single failed login | Review — Tier 1 multiplier may push score high | Low priority, likely noise |
| Process injection alert | Treat as critical, isolate if confirmed | Investigate, correlate with other events first |
| Auto-isolate triggered | Verify and notify L2 before unblocking | Analyst can unblock after FP confirmation |

**Rule of thumb:** on a Tier 1 host, assume compromise until proven otherwise. On Tier 3, assume noise until corroborated by a second event.

---

### Escalating to L2 — What That Means in This Lab

In this lab, L2 escalation means:

1. **Do not close the case.** Leave it open and assigned to yourself.
2. Add a TheHive task note summarising what you found, what you ruled out, and why you could not classify it.
3. Tag the case with `escalated-l2`.
4. Flag the following scenarios as mandatory L2 escalation regardless of score:
   - Lateral movement detected (multiple hosts, sequential logins).
   - Command-and-control suspected (`T1071`, outbound on unusual ports).
   - Data exfiltration suspected (`T1041`, large outbound transfer).
   - Auto-isolate fired on a Tier 1 host.
   - The analyst cannot classify the incident within 15 minutes.
5. In a production environment, L2 would also receive a Slack or email notification via Shuffle — configure the notify node in the TheHive Case Creation playbook to support this.

---

### Handling a Multi-Stage Attack Correlation

When the pipeline creates a case with multiple MITRE techniques (e.g., `T1110`, `T1059`, `T1071`):

1. The alert represents correlated events — not a single trigger. The `alert_count` field shows how many raw alerts were collapsed into this incident.
2. In Kibana, filter by `incident_id` to see the full timeline of contributing events.
3. Look for the progression: **Reconnaissance → Initial Access → Execution → C2**. If three or more stages are present, treat it as an active intrusion.
4. Do not close as FalsePositive unless you can explain every technique tag individually.
5. Escalate to L2 if the chain includes `T1059` (execution) plus any exfiltration or C2 technique.
