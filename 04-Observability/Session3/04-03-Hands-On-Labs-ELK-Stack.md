# Elastic Stack on Minikube - Hands-On Labs

**Stack:** Filebeat + Elasticsearch + Kibana  
**Goal:** Collect Kubernetes container logs, enrich and parse them, search in Kibana, build a dashboard, and trigger an alert.


## Prerequisites

- Docker Desktop or another Minikube-supported driver
- Minikube and `kubectl`
- At least 4 CPU and 6 GB RAM available to Minikube

```bash
minikube version
kubectl version --client
```

---

## Lab 1 - Deploy Elasticsearch and Kibana

### 1. Start Minikube

Using the same instructions in session 2

### 2. Deploy the stack

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-elasticsearch.yaml
kubectl apply -f manifests/02-kibana.yaml

kubectl rollout status deployment/elasticsearch -n logging --timeout=5m
kubectl rollout status deployment/kibana -n logging --timeout=5m
kubectl get pods -n logging
```

If a pod is not ready:

```bash
kubectl describe pod -n logging -l app=elasticsearch
kubectl logs -n logging deployment/elasticsearch --tail=50
```

### 3. Open Kibana

Keep this command running in a separate terminal:

```bash
kubectl port-forward -n logging service/kibana 5601:5601
```

Open `http://localhost:5601` and wait for the Kibana home page.

**Checkpoint:** Elasticsearch and Kibana are `Running`, and Kibana opens in the browser.

---

## Lab 2 - Parse and collect Kubernetes logs

### Before you begin: why ingest pipelines are needed

Kubernetes writes application output as container log lines. Filebeat can collect those lines and add pod, namespace, node and label information, but the application message may still be only one unstructured string:

```text
2026-08-16T15:10:00Z app[1234] ERROR Database connection timeout
```

An Elasticsearch ingest pipeline processes each event before it is indexed. In this lab:

- `plain-app-logs` uses Grok to extract fields such as `app.service`, `app.pid`, `log.level` and `app.message`.
- `json-app-logs` parses JSON and normalizes fields such as `log.level`, `app.msg`, `http.status_code` and `@timestamp`.

This makes precise searches such as `log.level: "error"` possible instead of searching for the word `ERROR` inside the complete message.

The complete flow is:

```text
Pod stdout -> container log -> Filebeat adds Kubernetes metadata
           -> workload label selects a pipeline
           -> Elasticsearch parses and indexes fields
           -> Kibana searches and visualizes those fields
```

The demo workloads use `log_format: plain` or `log_format: json`. Filebeat reads that Kubernetes label and routes the event to the matching ingest pipeline.

Creating the pipelines manually in Kibana is a learning exercise, not a production requirement. In a real environment, the same Elasticsearch API requests are normally executed by CI/CD, Terraform, Ansible, Elastic Fleet or a Kubernetes setup Job.

Create both pipelines before starting Filebeat. If Filebeat sends an event to a pipeline that does not exist, Elasticsearch rejects that event until the pipeline becomes available.

### 1. Create both ingest pipelines

In Kibana, open **Dev Tools → Console**. Run the request below, using the complete JSON body from `pipelines/plain-app-logs.json`:

```http
PUT _ingest/pipeline/plain-app-logs
```

Then run the second request with the JSON body from `pipelines/json-app-logs.json`:

```http
PUT _ingest/pipeline/json-app-logs
```

Confirm both pipelines:

```http
GET _ingest/pipeline/plain-app-logs,json-app-logs
```

### 2. Deploy the demo workloads and Filebeat

```bash
kubectl apply -f manifests/06-demo-apps.yaml
kubectl apply -f manifests/03-filebeat-rbac.yaml
kubectl apply -f manifests/04-filebeat-config.yaml
kubectl apply -f manifests/05-filebeat-daemonset.yaml

kubectl rollout status daemonset/filebeat -n logging --timeout=3m
kubectl get pods -n logging
kubectl get pods -n demo
kubectl logs -n logging daemonset/filebeat --tail=50
```

### 3. Verify documents

Run in Kibana Dev Tools:

```http
GET filebeat-*/_count
```

The count should increase as the demo applications generate logs.

**Checkpoint:** Filebeat runs as a DaemonSet. Events contain Kubernetes metadata and a normalized `log.level` field.

---

## Lab 3 - Discover logs with KQL

### 1. Create a data view

1. Open **Stack Management → Data Views**.
2. Select **Create data view**.
3. Name: `Kubernetes Logs`
4. Index pattern: `filebeat-*`
5. Timestamp field: `@timestamp`
6. Save the data view.

### 2. Explore in Discover

Open **Discover**, select `Kubernetes Logs`, and set the time picker to **Last 15 minutes**.

Add these columns: `@timestamp`, `kubernetes.namespace`, `kubernetes.pod.name`, `log.level`, and either `app.message` or `app.msg`.

Run these KQL filters one at a time:

```text
kubernetes.namespace: "demo"
```

```text
kubernetes.labels.app: "plain-logger"
```

```text
kubernetes.namespace: "demo" and log.level: ("warn" or "error")
```

```text
kubernetes.labels.log_format: "json" and app.http.status_code >= 400
```

Open one document and locate the original message, Kubernetes metadata added by Filebeat, and fields created by the ingest pipeline.


---

## Lab 4 - Build a Kibana dashboard

Create a dashboard named **Kubernetes Log Overview** with four Lens panels:

1. **Log events over time** - line chart; `@timestamp` and `Count`.
2. **Logs by level** - bar or donut chart; top values of `log.level`.
3. **Top applications** - bar chart; top values of `kubernetes.labels.app`.
4. **Error events** - metric; `Count` with panel filter `log.level: "error"`.

Save it, select **Last 15 minutes**, and apply this dashboard filter:

```text
kubernetes.namespace: "demo"
```

**Checkpoint:** The dashboard shows log volume, noisy applications, and current error count.

---

## Final verification

```bash
kubectl get pods -n logging
kubectl get pods -n demo
kubectl get daemonset filebeat -n logging
```

You should now have centralized and enriched Kubernetes logs, parsed fields, a saved dashboard, and a working error-burst alert.

## Cleanup

```bash
minikube delete -p logging
```
