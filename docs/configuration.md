# Configuration Reference

All key configuration files and environment variables across the four VMs.

---

## vm-ai — Environment Variables (`/opt/soc-ai/.env`)

| Variable | Description | Example |
|---|---|---|
| `ES_HOST` | Elasticsearch URL | `http://192.168.100.20:9200` |
| `ES_USER` | Elasticsearch username | `elastic` |
| `ES_PASSWORD` | Elasticsearch password | set during Phase 5 |
| `REDIS_HOST` | Redis host | `127.0.0.1` |
| `REDIS_PORT` | Redis port | `6379` |
| `REDIS_PASSWORD` | Redis auth password | set during Phase 5 |
| `DB_HOST` | PostgreSQL host | `127.0.0.1` |
| `DB_NAME` | PostgreSQL database | `soc_db` |
| `DB_USER` | PostgreSQL user | `soc_user` |
| `DB_PASSWORD` | PostgreSQL password | set during Phase 5 |
| `THEHIVE_URL` | TheHive API base URL | `http://192.168.100.40:9000` |
| `THEHIVE_API_KEY` | TheHive API key | set during Phase 5 |
| `SHUFFLE_WEBHOOK_URL` | Shuffle webhook endpoint | `http://192.168.100.40:3001/api/v1/hooks/...` |
| `OLLAMA_MODEL` | Local LLM model name | `phi3` |
| `OLLAMA_TIMEOUT` | LLM request timeout (seconds) | `300` |

---

## vm-ai — Pipeline Thresholds (`main_pipeline.py`)

| Parameter | Default | Effect |
|---|---|---|
| `POLL_INTERVAL` | `30` seconds | How often the pipeline queries Elasticsearch for new alerts |
| `HIGH_RISK_THRESHOLD` | `70` | Minimum score to create a TheHive case |
| `CRITICAL_THRESHOLD` | `90` | Minimum score to trigger auto-isolation |
| `DEDUP_TTL` | `3600` seconds | Redis fingerprint expiry window |

---

## vm-ai — Asset Criticality Tiers (`asset_graph.py`)

| Tier | Multiplier | Hosts in Lab |
|---|---|---|
| Tier 1 — Critical | `1.5×` | `vm-collection` (Wazuh Manager), `vm-siem` (SIEM) |
| Tier 2 — Important | `1.2×` | `vm-ai` (AI Engine), `vm-response` (TheHive/Shuffle) |
| Tier 3 — Standard | `1.0×` | `WIN-ENDPOINT-01`, `WIN-ENDPOINT-02`, workstations |

To add a new host, edit `asset_graph.py`:

```python
ASSET_GRAPH = {
    "192.168.100.10": {"hostname": "vm-collection", "role": "wazuh_manager", "tier": 1},
    "192.168.100.20": {"hostname": "vm-siem",       "role": "siem",          "tier": 1},
    "192.168.100.30": {"hostname": "vm-ai",          "role": "ai_engine",     "tier": 2},
    "192.168.100.40": {"hostname": "vm-response",    "role": "response",      "tier": 2},
    # Add new hosts below
}
```

---

## vm-ai — MITRE Mapping (`mitre_mapper.py`)

Key mapping dictionaries:

```python
RULE_TO_MITRE = {
    "5710": ["T1110"],   # SSH brute force
    "5712": ["T1110"],
    "5503": ["T1548"],   # Privilege escalation
    "5715": ["T1110"],
    "31103": ["T1040"],  # Network sniffing
    "31151": ["T1190"],  # Exploit public-facing application
    "200": ["T1595"],    # Active scanning
    "210": ["T1046"],    # Network service discovery
    "591": ["T1055"],    # Process injection
    "5902": ["T1078"],   # Valid accounts
}
```

To add a new mapping: find the Wazuh rule ID in Kibana, add the entry, then `sudo systemctl restart soc-pipeline`.

---

## vm-siem — Elasticsearch (`/etc/elasticsearch/elasticsearch.yml`)

```yaml
cluster.name: soc-lab
node.name: vm-siem
network.host: 0.0.0.0
http.port: 9200
xpack.security.enabled: true
xpack.security.http.ssl.enabled: false
discovery.type: single-node
```

JVM heap (set in `/etc/elasticsearch/jvm.options.d/heap.options`):

```
-Xms8g
-Xmx8g
```

---

## vm-siem — System Tuning

```bash
# Required for Elasticsearch — set in /etc/sysctl.conf
vm.max_map_count=262144

# Disable swap
sudo swapoff -a
```

---

## vm-response — TheHive Docker Startup

```bash
docker start cassandra
sleep 30
docker start elasticsearch
sleep 30
docker start thehive
# Wait 3–5 minutes before accessing http://192.168.100.40:9000
```

Default analyst account after Phase 5: `analyst@thehive.local` with rotated password stored in your private setup notes.
