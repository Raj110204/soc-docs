# Detection & Rules

The platform combines host-based, network-based, and endpoint telemetry into one normalized alert pipeline.

## Alert Sources

| Source | Example Signal | MITRE Use |
| --- | --- | --- |
| Wazuh | SSH brute force, privilege escalation, file integrity | Rules map to techniques such as `T1110` and `T1548` |
| Suricata | Port scans, suspicious outbound traffic, exploit signatures | Network behavior maps to discovery and command-and-control techniques |
| Sysmon | Process creation, PowerShell, registry and network events | Execution and persistence visibility |
| auditd / rsyslog | Linux auth and syscall events | Linux host investigation context |

## MITRE Mapping Workflow

1. Query Kibana for alerts missing `mitre_techniques`.
2. Identify the Wazuh rule ID or Suricata signature.
3. Add the mapping to `RULE_TO_MITRE` in `mitre_mapper.py`.
4. Restart the pipeline.
5. Validate coverage in Kibana.

## Mapped Rules

The active `RULE_TO_MITRE` dictionary covers the following Wazuh rule IDs and MITRE techniques:

| Rule ID | Source | MITRE Technique | Description |
|---|---|---|---|
| 5710 | Wazuh | T1110 | SSH brute force |
| 5712 | Wazuh | T1110 | SSH brute force (variant) |
| 5715 | Wazuh | T1110 | SSH authentication failure |
| 5503 | Wazuh | T1548 | Privilege escalation attempt |
| 5902 | Wazuh | T1078 | Valid account misuse |
| 31103 | Wazuh | T1040 | Network sniffing |
| 31151 | Wazuh | T1190 | Exploit public-facing application |
| 200 | Suricata | T1595 | Active scanning |
| 210 | Suricata | T1046 | Network service discovery |
| 591 | Suricata | T1055 | Process injection |

To add or modify mappings, see the full dictionary in [Configuration Reference](configuration.md#vm-ai-mitre-mapping-mitre_mapperpy).

!!! tip "Finding Unmapped Rules"
    Run this in Kibana Dev Tools to find rule IDs with no MITRE tag:

```json
GET soc-alerts-*/_search
{
  "query": { "bool": { "must_not": { "exists": { "field": "mitre_techniques" } } } },
  "aggs": { "by_rule": { "terms": { "field": "rule.id", "size": 20 } } }
}
```

## Validation Test Cases

| Test | Input | Expected Output | Result |
| --- | --- | --- | --- |
| SSH brute force | 20 failed SSH logins | Wazuh rule 5710 and `T1110` | PASS |
| Network scan | Nmap scan against collection VM | Suricata alert indexed | PASS |
| Deduplication | 500 identical brute-force alerts | One incident with `alert_count=500` | PASS |
| Critical playbook | Incident score `>= 90` | Wazuh block plus TheHive critical case | PASS |
| Multi-stage correlation | Brute force + PowerShell + outbound | One incident with `T1110`, `T1059`, `T1071` | PASS |

## Coverage Result

By Phase 3, MITRE ATT&CK coverage reached 83 percent of observed alert types, exceeding the 80 percent target used in the implementation guide.
