# Troubleshooting

Common issues encountered during lab setup and operation, with confirmed fixes.

---

## Pipeline Not Polling Alerts

**Symptom:** `journalctl -u soc-pipeline` shows no new alerts being processed despite Kibana showing new documents.

**Cause:** Usually a stale Elasticsearch query offset or authentication failure after credential rotation.

**Fix:**

```bash
# On vm-ai
sudo systemctl restart soc-pipeline
journalctl -u soc-pipeline -f
```

If the error mentions `AuthorizationException` or `401`, update credentials in `main_pipeline.py` and `.env` to match the current `elastic` password.

---

## TheHive Returns 503 or Blank Page

**Symptom:** Browser shows 503 or a blank white page at `http://192.168.100.40:9000`.

**Cause:** TheHive started before Cassandra or its internal Elasticsearch was ready.

**Fix:**

```bash
# On vm-response — check container startup order
docker ps -a
# Restart in correct order
docker restart cassandra
sleep 30
docker restart elasticsearch
sleep 30
docker restart thehive
# Wait 3–5 minutes before opening the browser
```

---

## Redis Authentication Failure

**Symptom:** Pipeline logs show `NOAUTH Authentication required`.

**Cause:** `requirepass` was set in `redis.conf` but `REDIS_PASSWORD` in `.env` was not updated.

**Fix:**

```bash
# On vm-ai
grep REDIS_PASSWORD /opt/soc-ai/.env
# Update if wrong
sed -i 's/REDIS_PASSWORD=.*/REDIS_PASSWORD=YourNewPassword/' /opt/soc-ai/.env
sudo systemctl restart soc-pipeline
```

---

## Shuffle Webhook Not Triggering

**Symptom:** Pipeline logs show `POST to Shuffle returned 404` or no Shuffle execution appears in the dashboard.

**Cause:** Webhook ID changed after a Shuffle update or workflow was deactivated.

**Fix:**

1. Open Shuffle at `http://192.168.100.40:3001`.
2. Open the target workflow → click the webhook trigger node.
3. Copy the current webhook URL.
4. Update `SHUFFLE_WEBHOOK_URL` in `/opt/soc-ai/.env` on `vm-ai`.
5. Restart the pipeline.

---

## Wazuh Agent Shows Disconnected

**Symptom:** `agent_control -l` on `vm-collection` shows an agent as `Disconnected`.

**Fix:**

```bash
# On the affected agent VM
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent

# On vm-collection — confirm re-registration
/var/ossec/bin/agent_control -l
```

If the agent still shows disconnected after 60 seconds, check that UFW on `vm-collection` allows port `1514/udp` from the agent IP.

---

## Elasticsearch Cluster Red / Yellow Health

**Symptom:** `curl http://192.168.100.20:9200/_cluster/health` returns `red`.

**Common causes and fixes:**

```bash
# Check unassigned shards
curl -u elastic:YOUR_PASSWORD http://192.168.100.20:9200/_cat/shards?v | grep UNASSIGNED

# Force shard allocation (single-node lab)
curl -u elastic:YOUR_PASSWORD -X PUT http://192.168.100.20:9200/_settings \
  -H "Content-Type: application/json" \
  -d '{"index.number_of_replicas": 0}'
```

In a single-node lab, `yellow` health is expected and normal — it means replicas are unassigned because there is only one node. `green` is not required for operation.

---

## XGBoost Model Not Updating After Retrain

**Symptom:** `retrain.py` runs successfully but live scoring still reflects old scores.

**Fix:** The pipeline loads the model at startup. Restart the service after retraining:

```bash
python /home/vboxuser/soc-ai/retrain.py
sudo systemctl restart soc-pipeline
```

---

## Kibana Shows No Data / Empty Dashboard

**Symptom:** SOC dashboard panels show "No data" despite alerts being indexed.

**Fix:**

1. Open Kibana → Stack Management → Index Patterns.
2. Confirm `soc-alerts-*` index pattern exists and the time field is `@timestamp`.
3. Set the Kibana time picker to `Last 24 hours` or `Last 7 days`.
4. If the index pattern is missing, recreate it against the `soc-alerts-*` wildcard.
