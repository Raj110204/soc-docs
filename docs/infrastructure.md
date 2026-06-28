# Infrastructure

The infrastructure layer turns raw endpoint and network telemetry into searchable SOC data. It is split between `vm-collection` and `vm-siem`.

## Collection Layer

| Component | Purpose |
| --- | --- |
| Wazuh Manager | Host-based intrusion detection and agent management |
| Wazuh Agents | Endpoint telemetry and security event forwarding |
| Suricata | Network IDS alert generation through EVE JSON |
| Sysmon | Windows process, network, registry, and file event telemetry |
| auditd / rsyslog | Linux authentication and syscall visibility |
| Filebeat | Ships normalized logs to Logstash |

## SIEM Layer

Logstash receives events on port `5044`, applies source-specific parsing, and writes normalized documents into Elasticsearch under the `soc-alerts-*` index pattern. Kibana provides the analyst-facing search and dashboard layer.

## Hardening Controls

Phase 5 added the production-readiness controls documented in the report:

- Default credentials rotated across TheHive, Shuffle, PostgreSQL, Redis, and Elasticsearch.
- Elasticsearch `xpack.security` enabled.
- Kibana and TheHive exposed through Nginx TLS reverse proxies.
- API keys moved into environment configuration with restricted file permissions.
- UFW rules scoped inter-VM traffic to the SOC subnet.
- Fail2ban enabled for SSH brute-force protection.
- Clean `Phase5-Hardened-Clean` VM snapshots created after verification.

## Operational Notes

Elasticsearch required JVM heap tuning on `vm-siem` during high-volume ingestion. The stable setting used in testing was an 8 GB heap on a 16 GB VM, paired with `vm.max_map_count=262144` and swap disabled.

## Logstash Pipeline Configuration

The core Logstash pipeline config lives at `/etc/logstash/conf.d/soc-pipeline.conf` on `vm-siem`.

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [agent][type] == "filebeat" {
    mutate {
      add_field => { "alert_source" => "wazuh" }
    }
  }
  if [rule][id] {
    mutate {
      add_field => { "alert_type" => "%{[rule][description]}" }
    }
  }
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://192.168.100.20:9200"]
    user => "elastic"
    password => "${ES_PASSWORD}"
    index => "soc-alerts-%{+YYYY.MM.dd}"
  }
}
```

## Filebeat Setup on vm-collection

Filebeat ships Wazuh alerts and Suricata EVE JSON to Logstash. Key config at `/etc/filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/ossec/logs/alerts/alerts.json
    fields:
      alert_source: wazuh
    json.keys_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/suricata/eve.json
    fields:
      alert_source: suricata
    json.keys_under_root: true

output.logstash:
  hosts: ["192.168.100.20:5044"]
```

After any config change, restart with:

```bash
sudo systemctl restart filebeat
sudo systemctl status filebeat
```

## Service Startup Order

Always start services in this order to avoid connection failures:

| Step | VM | Command | Wait |
|---|---|---|---|
| 1 | vm-siem | `systemctl start elasticsearch` | 30 seconds |
| 2 | vm-siem | `systemctl start logstash kibana` | 20 seconds |
| 3 | vm-response | Start Cassandra container | 30 seconds |
| 4 | vm-response | Start Elasticsearch container | 30 seconds |
| 5 | vm-response | Start TheHive container | 3–5 minutes |
| 6 | vm-response | `systemctl start shuffle` | 10 seconds |
| 7 | vm-collection | `systemctl start wazuh-manager filebeat` | 15 seconds |
| 8 | vm-ai | `systemctl start redis-server postgresql soc-pipeline` | 10 seconds |

!!! warning "TheHive Startup"
    TheHive requires Cassandra and its internal Elasticsearch to be fully healthy before it starts. Rushing this causes a blank login page or 503 error. Wait the full 3–5 minutes after starting the TheHive container.
