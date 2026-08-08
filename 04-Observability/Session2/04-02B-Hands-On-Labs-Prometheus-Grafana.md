# Session 2: Prometheus and Grafana on Minikube

## Overview

In this session, you will build a small Kubernetes monitoring stack and follow the complete metrics path:

`Kubernetes targets -> Prometheus -> PromQL -> Grafana dashboards -> Grafana alert`

**Platform:** Minikube  
**Focus:** Metrics, dashboards, and one tested alert

## What you will build

- Prometheus with Kubernetes service discovery
- Node Exporter for node operating-system metrics
- kube-state-metrics for Kubernetes object state
- Grafana connected to Prometheus
- An imported Node Exporter dashboard and a four-panel Kubernetes dashboard
- A Grafana alert observed through its complete state lifecycle

---

## Lab 1: Install & Start the Minikube cluster

**Objective:** Install Minikube and kubectl on your local machine. Verify the cluster starts successfully and you can communicate with it.

### Step 1: Install kubectl

**macOS:**
```bash
# Using Homebrew
brew install kubectl

# Verify
kubectl version --client
```

**Linux (Ubuntu/Debian):**
```bash
# Download latest stable
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

**Windows (PowerShell as Administrator):**
```powershell
# Using Chocolatey
choco install kubernetes-cli

# Or download directly
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe to a directory in your PATH

# Verify
kubectl version --client
```

### Step 2: Install Minikube

Minikube creates a single-node Kubernetes cluster on your local machine using Docker as the driver.

**macOS:**
```bash
brew install minikube
```

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```


**Windows (PowerShell as Administrator):**
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
choco install minikube

# Or download installer from: https://minikube.sigs.k8s.io/docs/start/
```

### Step 3: Start Your First Cluster

```bash
# Start Minikube with Docker driver
minikube start --driver=docker

# If Minikube is running on Windows with HyperV use below command
# Enable Hyper-V:
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Restart your computer
minikube start --driver=hyperv --container-runtime=containerd

# You should see output like:
# Done! kubectl is now configured to use "minikube" cluster
```

### Step 4: Verify Everything Works

```bash
# Check Minikube status
minikube status
# Expected: host: Running, kubelet: Running, apiserver: Running

# Check cluster info
kubectl cluster-info
# Expected: Kubernetes control plane is running at https://...

# Check the node
kubectl get nodes
# Expected: minikube   Ready   control-plane   ...

# Check system pods (these run Kubernetes itself)
kubectl get pods -n kube-system
# You should see: coredns, etcd, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler, storage-provisioner
```
---

## Lab 2: Deploy Prometheus and metric sources

### Objective

Deploy the monitoring components and verify that Prometheus can discover and scrape Kubernetes targets.

### Step 1: Apply the core manifests

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-prometheus-rbac.yaml
kubectl apply -f manifests/02-prometheus-config.yaml
kubectl apply -f manifests/03-prometheus.yaml
kubectl apply -f manifests/04-node-exporter.yaml
kubectl apply -f manifests/05-kube-state-metrics.yaml
kubectl apply -f manifests/06-grafana.yaml
```

### Step 2: Wait for the workloads

```bash
kubectl get pods -n monitoring -w
```

Press `Ctrl+C` when the Prometheus, Node Exporter, kube-state-metrics, and Grafana pods are `Running` and ready.

Verify the Kubernetes objects:

```bash
kubectl get all -n monitoring
kubectl get configmap,serviceaccount -n monitoring
```

### Step 3: Open Prometheus

Keep this command running in a separate terminal:

```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```

Open `http://localhost:9090`.

### Step 4: Inspect service discovery

In Prometheus, open **Status -> Target health**.

Locate these jobs:

- `prometheus`
- `kubernetes-apiservers`
- `kubernetes-nodes`
- `kubernetes-cadvisor`
- `kube-state-metrics`
- `node`
- `kubernetes-pods`

Discover what each source answers:

| Source | Question it answers |
|---|---|
| Node Exporter | What is happening on the node operating system? |
| cAdvisor | What resources are containers consuming? |
| kube-state-metrics | What state does Kubernetes report for its objects? |
| Application `/metrics` | What is happening inside the application? |

### Step 5: Connect discovery to the configuration

Open `manifests/02-prometheus-config.yaml` and locate:

- `kubernetes_sd_configs`
- `role: node`, `role: pod`, and `role: endpoints`
- `relabel_configs`
- `prometheus.io/scrape`

Prometheus first discovers a broad list of Kubernetes objects. Relabel rules decide which targets remain and how Prometheus reaches them.

### Success criteria

- All monitoring pods are ready.
- Prometheus opens in the browser.
- The important targets show `UP`.
- You can explain why Node Exporter and kube-state-metrics are not interchangeable.

---

## Lab 3: Query live metrics with PromQL

### Objective

Use a small set of PromQL patterns that answer real cluster-health questions.

Open the Prometheus **Query** page. Run each query in **Table** and **Graph** view.

### Step 1: Check target health

```promql
up
```

`1` means the latest scrape succeeded. `0` means Prometheus knows the target but cannot scrape it.

Summarize by job:

```promql
sum by (job) (up)
```

### Step 2: Calculate node CPU utilization

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Why `rate()`? `node_cpu_seconds_total` is a counter. Its raw value only increases, so the useful signal is its rate of change.

### Step 3: Calculate node memory utilization

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

Memory values are gauges. They move up and down and can be read directly.

### Step 4: Inspect Kubernetes object state

```promql
sum by (phase) (kube_pod_status_phase)
```

Only keep active series:

```promql
sum by (phase) (kube_pod_status_phase == 1)
```

### Step 5: Find the busiest pods

```promql
topk(5,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
  )
)
```

### Success criteria

- You can filter and aggregate time series by labels.
- You know when to use `rate()`.
- You can distinguish node, container, and Kubernetes object metrics.

---

## Lab 4: Connect Grafana and build a dashboard

### Objective

Connect Grafana to Prometheus using Kubernetes service DNS and build a concise cluster dashboard.

### Step 1: Open Grafana

Keep this command running in a separate terminal:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```

Open `http://localhost:3000` and sign in:

- Username: `admin`
- Password: `admin123`

### Step 2: Add Prometheus as a data source

1. Open **Connections -> Data sources**.
2. Choose **Add new data source -> Prometheus**.
3. Set the server URL to:

```text
http://prometheus.monitoring.svc.cluster.local:9090
```

4. Set the scrape interval to `15s` if the option is displayed.
5. Choose **Save & test**.

Grafana runs inside Kubernetes, so it connects to the Prometheus Service through cluster DNS. `localhost:9090` would point back to the Grafana container.

### Step 3: Verify the connection with Explore

1. Open **Explore**.
2. Select the Prometheus data source.
3. Run:

```promql
sum by (job) (up)
```

### Step 4: Import the Node Exporter Full dashboard

Grafana has a public dashboard library. Import dashboard `1860` to quickly obtain a detailed Node Exporter view:

1. Open **Dashboards -> New -> Import**.
2. Enter dashboard ID `1860` and choose **Load**.
3. Select your Prometheus data source.
4. Choose **Import**.
5. Open the **Job** or **Node** variable and select the available Minikube target if required.

Dashboard `1860` is **Node Exporter Full**. It visualizes Linux node CPU, memory, disk, filesystem and network metrics. The Prometheus manifest in this lab uses the job name `node`, which matches the dashboard's expected default job name.

> Some panels may show `N/A` when a collector or operating-system feature is unavailable inside Minikube. This is expected; focus on CPU, memory, filesystem and network panels.

### Step 5: Create a small dashboard manually

Open **Dashboards -> New -> New dashboard**. Add these four panels:

| Panel | Visualization | Query | Unit |
|---|---|---|---|
| Scrape Targets Up | Stat | `sum(up)` | short |
| Running Pods | Stat | `sum(kube_pod_status_phase{phase="Running"} == 1)` | short |
| Node CPU Utilization | Time series | `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | percent (0-100) |
| Top Pods by CPU | Time series | `topk(5, sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!="",image!=""}[5m])))` | cores |

For each panel:

1. Choose **Add visualization**.
2. Select Prometheus.
3. Switch to the code editor if required and paste the query.
4. Set the panel title, visualization, and unit.
5. Choose **Apply**.

Save the dashboard as `Minikube Cluster Overview`.

### Success criteria

- Grafana successfully queries Prometheus.
- Dashboard `1860` imports and displays Node Exporter metrics.
- The dashboard contains four readable panels.
- Each panel answers one operational question.

---

## Lab 5: Create, trigger, and resolve a Grafana alert

### Objective

Create one useful Kubernetes alert and observe the complete alert lifecycle in Grafana. Notification integrations are intentionally outside this session.

### Step 1: Create the alert rule

Open **Alerting -> Alert rules -> New alert rule**.

Configure:

- **Rule name:** `KubePodStuckWaiting`
- **Data source:** Prometheus
- **Query A:**

```promql
sum(kube_pod_container_status_waiting_reason{reason=~"ImagePullBackOff|ErrImagePull|CrashLoopBackOff"}) or vector(0)
```

- **Condition:** Query value is above `0`
- **Evaluation group:** Create `lab-evaluation`, every `1m`
- **Pending period:** `1m`
- **Label:** `severity = warning`
- **Summary:** `A pod is stuck in a waiting state`

Save the rule. It should initially be `Normal`.

### Step 2: Trigger the alert deliberately

The lab workload includes a deliberately invalid image and a small CPU burner:

```bash
kubectl apply -f manifests/07-lab-workloads.yaml
kubectl get pods -n lab-workloads -w
```

Press `Ctrl+C` after `broken-app` shows `ImagePullBackOff` or `ErrImagePull`.

In Grafana, watch the rule move through:

`Normal -> Pending -> Firing`

### Step 3: Resolve the alert

```bash
kubectl delete namespace lab-workloads
```

After the next evaluation cycles, the alert returns to `Normal`.

### Success criteria

- The rule moves through `Normal`, `Pending`, and `Firing`.
- Removing the bad workload resolves the alert.
- You can explain why the pending period prevents noisy alerts.

---

## Final verification

- [ ] Minikube profile `observability` is running
- [ ] Prometheus, Grafana, Node Exporter, and kube-state-metrics are ready
- [ ] Important Prometheus targets are `UP`
- [ ] PromQL queries return node, container, and object-state metrics
- [ ] Grafana reaches Prometheus through Kubernetes service DNS
- [ ] Dashboard `1860` displays Node Exporter metrics
- [ ] The dashboard contains four working panels
- [ ] The alert was observed in `Pending`, `Firing`, and resolved states

## Cleanup

The quickest cleanup removes the complete lab cluster:

```bash
minikube delete --profile observability
```

If you want to keep Minikube but remove only this stack:

```bash
kubectl delete namespace lab-workloads --ignore-not-found
kubectl delete namespace monitoring
kubectl delete clusterrole prometheus kube-state-metrics
kubectl delete clusterrolebinding prometheus kube-state-metrics
```

## Key takeaways

1. Prometheus pulls metrics from discovered targets and stores labeled time series.
2. Node Exporter, cAdvisor, kube-state-metrics, and application metrics answer different questions.
3. Use `rate()` for counters; read gauges directly.
4. Grafana queries Prometheus and turns PromQL into dashboards and alerts.
5. Validate an alert by deliberately triggering it and observing both firing and recovery.
