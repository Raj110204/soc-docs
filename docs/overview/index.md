# Project Overview

Modern SOC teams are not only fighting attackers; they are fighting alert volume. Traditional SIEM rules create thousands of alerts, many of them duplicated or low value, and L1 analysts must still determine which ones deserve immediate attention.

The Self-Calibrating SOC Platform addresses that problem with an open-source architecture that learns from analyst decisions. Instead of keeping severity static, it uses environment-specific feedback to improve alert prioritization over time.

## Problem Statement

Static SOC pipelines create three operational failures:

- **Alert volume overload:** repeated brute-force or scan alerts are reviewed individually instead of as one incident.
- **Multi-stage attack blindness:** failed login, execution, and outbound traffic events are often shown as unrelated alerts.
- **Alert fatigue:** analysts become desensitized when the queue is dominated by repeated false positives.

Commercial platforms solve parts of this problem, but often cost tens or hundreds of thousands of dollars per year. This project demonstrates that similar core capabilities can be built using open-source components and a focused Python AI layer.

## Objectives

- Ingest alerts from HIDS, network IDS, endpoint, and Linux telemetry sources.
- Deduplicate repeated alerts using incident fingerprints.
- Map alerts to MITRE ATT&CK techniques and correlate related behavior.
- Score incidents with XGBoost using alert count, asset tier, technique weight, and context.
- Generate analyst-friendly LLM guidance for each high-risk incident.
- Create TheHive cases and trigger SOAR actions through Shuffle.
- Store analyst outcomes and retrain the model weekly.

## Scope

| In Scope | Out of Scope |
| --- | --- |
| Four-VM Ubuntu lab architecture | Full Active Directory deployment |
| Wazuh, Suricata, Sysmon, auditd telemetry | Cloud-native workload monitoring |
| ELK ingestion, normalization, and dashboards | Production multi-node Elasticsearch cluster |
| XGBoost scoring and feedback retraining | Automatic local LLM failover |
| TheHive and Shuffle SOAR automation | MISP as a required initial dependency |
