# Prometheus Configuration on Kubernetes

The easiest way to understand this configuration is to start with one simple idea:

> **Prometheus is a program that repeatedly asks different components: “Give me your metrics.”**

This `ConfigMap` tells Prometheus:

> **what to monitor, how to find it, and from where to collect the metrics.**

You can understand the complete file in four main parts.

---

# 1. Kubernetes ConfigMap Wrapper

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
```

This part is **Kubernetes configuration**, not Prometheus configuration yet.

We are creating a ConfigMap called:

```text
prometheus-config
```

inside the:

```text
monitoring
```

namespace.

The purpose of this ConfigMap is to store the Prometheus configuration file.

Then:

```yaml
data:
  prometheus.yml: |
```

means:

> Store a file named `prometheus.yml` inside this ConfigMap.

Everything below this point is the actual Prometheus configuration.

---

# 2. Global Settings

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: minikube
```

## `scrape_interval`

```yaml
scrape_interval: 15s
```

This means:

> Prometheus collects metrics from its targets every 15 seconds.

For example:

```text
Prometheus → Node Exporter → "Give me CPU metrics"

Prometheus → kube-state-metrics → "Give me Kubernetes object metrics"

Prometheus → Application → "Give me application metrics"
```

Prometheus repeats this process every 15 seconds.

---

## `evaluation_interval`

```yaml
evaluation_interval: 15s
```

Prometheus can also evaluate alerting and recording rules.

This setting means:

> Evaluate those rules every 15 seconds.

For example:

```text
Is CPU usage greater than 90%?

Is a pod unavailable?

Is application error rate too high?
```

---

## `external_labels`

```yaml
external_labels:
  cluster: minikube
```

This adds an extra label identifying the cluster.

For example, if later you monitor multiple Kubernetes clusters:

```text
cluster="minikube"

cluster="dev"

cluster="production"
```

you can easily identify where the metrics came from.

---

# 3. `scrape_configs`

This is the main section of the Prometheus configuration:

```yaml
scrape_configs:
```

Every `job_name` below represents something Prometheus wants to monitor.

In this configuration there are seven jobs:

```text
Prometheus
   |
   +-- prometheus
   +-- kubernetes-apiservers
   +-- kubernetes-nodes
   +-- kubernetes-cadvisor
   +-- kubernetes-pods
   +-- kube-state-metrics
   +-- node
```

Each job collects a different type of information.

---

# 4. Job 1 — Prometheus Monitoring Itself

```yaml
- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']
```

This is the simplest job and a good place to start.

Prometheus itself exposes metrics on port:

```text
9090
```

So Prometheus can monitor itself.

Conceptually:

```text
Prometheus
    |
    | GET /metrics
    v
localhost:9090
```

Prometheus exposes metrics such as:

```text
prometheus_http_requests_total

prometheus_tsdb_*

prometheus_engine_*
```

These help us understand the health and performance of Prometheus itself.

---

## Why `static_configs`?

```yaml
static_configs:
```

Here, we already know exactly where the target is:

```text
localhost:9090
```

Therefore, Kubernetes Service Discovery is not required.

---

# 5. Job 2 — Kubernetes API Server

```yaml
- job_name: 'kubernetes-apiservers'
```

This job monitors the Kubernetes API Server.

Instead of manually providing the API Server address, Prometheus asks Kubernetes:

> What endpoints exist in this cluster?

This is called **Service Discovery**.

```yaml
kubernetes_sd_configs:
  - role: endpoints
```

`sd` stands for:

```text
Service Discovery
```

So:

```yaml
role: endpoints
```

means:

> Discover Kubernetes Service endpoints.

---

## Why is filtering required?

A Kubernetes cluster may contain many Services:

```text
nginx

mysql

redis

grafana

prometheus

kubernetes
```

Prometheus only wants the Kubernetes API Server for this job.

That is why this configuration uses:

```yaml
relabel_configs:
```

For beginners, think of relabeling as:

> **Filtering and modifying discovered targets.**

This configuration:

```yaml
- source_labels:
    - __meta_kubernetes_namespace
    - __meta_kubernetes_service_name
    - __meta_kubernetes_endpoint_port_name
  action: keep
  regex: default;kubernetes;https
```

basically means:

> Discover everything, but keep only the Kubernetes API Server.

Conceptually:

```text
Discovered endpoints

nginx
mysql
redis
grafana
kubernetes
prometheus

       ↓ Filter

kubernetes
```

---

## HTTPS and Authentication

The Kubernetes API Server uses HTTPS:

```yaml
scheme: https
```

Prometheus also needs authentication.

It uses the Kubernetes ServiceAccount credentials mounted inside the Prometheus pod:

```yaml
tls_config:
  ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt

bearer_token_file:
  /var/run/secrets/kubernetes.io/serviceaccount/token
```

A simple explanation for learners is:

> Prometheus uses its Kubernetes ServiceAccount token to securely communicate with the Kubernetes API Server.

---

# 6. Job 3 — Kubelet Metrics

```yaml
- job_name: 'kubernetes-nodes'
```

Every Kubernetes node runs a component called:

```text
kubelet
```

The kubelet manages the pods running on that node.

For example:

```text
Kubernetes Node
|
+-- kubelet
|
+-- Pod A
+-- Pod B
+-- Pod C
```

Kubelet exposes operational metrics.

Prometheus discovers the nodes using:

```yaml
kubernetes_sd_configs:
  - role: node
```

---

## Accessing Kubelet Through the API Server

Instead of connecting directly to each node, this configuration sends the request through the Kubernetes API Server.

```yaml
- target_label: __address__
  replacement: kubernetes.default.svc:443
```

This means:

> Send the request to the Kubernetes API Server.

Then:

```yaml
replacement: /api/v1/nodes/${1}/proxy/metrics
```

means:

> Ask the API Server to proxy the request to the kubelet metrics endpoint of that node.

Architecture:

```text
Prometheus
     |
     v
Kubernetes API Server
     |
     v
Kubelet
     |
     v
/metrics
```

Instead of:

```text
Prometheus → Node:10250
```

This is useful when Prometheus does not have direct network access to kubelet port `10250`.

---

# 7. Job 4 — cAdvisor Metrics

```yaml
- job_name: 'kubernetes-cadvisor'
```

cAdvisor provides **container resource usage metrics**.

Examples include:

```text
Container CPU

Container memory

Container network

Container filesystem
```

Typical metrics include:

```text
container_cpu_usage_seconds_total

container_memory_working_set_bytes

container_network_receive_bytes_total
```

The metrics path is:

```yaml
replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

So the flow becomes:

```text
Prometheus
     |
     v
Kubernetes API Server
     |
     v
Node / Kubelet
     |
     v
cAdvisor Metrics
```

An easy way to explain cAdvisor is:

> **cAdvisor tells you what containers are consuming.**

For example:

```text
nginx pod is using 200 MB RAM

mysql pod is using 1.5 GB RAM

application pod is using 40% CPU
```

---

# 8. Job 5 — Application Pod Monitoring

```yaml
- job_name: 'kubernetes-pods'
```

This section allows application teams to tell Prometheus:

> Please monitor my pod.

They do this using Kubernetes annotations.

For example:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

Imagine an application:

```text
my-java-app
     |
     +-- port 8080
     |
     +-- /metrics
```

Prometheus discovers the pod automatically.

---

## `prometheus.io/scrape`

```yaml
- source_labels:
    - __meta_kubernetes_pod_annotation_prometheus_io_scrape
  action: keep
  regex: "true"
```

This means:

> Only scrape pods where `prometheus.io/scrape="true"`.

For example:

```text
Pod A: scrape=true   → Monitor

Pod B: scrape=false  → Ignore

Pod C: no annotation → Ignore
```

---

## `prometheus.io/path`

The annotation:

```yaml
prometheus.io/path: "/metrics"
```

tells Prometheus where the application's metrics are exposed.

Usually:

```text
/metrics
```

---

## `prometheus.io/port`

The annotation:

```yaml
prometheus.io/port: "8080"
```

tells Prometheus which port contains the metrics endpoint.

Prometheus may eventually call something like:

```text
http://10.244.1.25:8080/metrics
```

This means application teams do not have to modify `prometheus.yml` whenever a new application is deployed.

They simply annotate their pods.

---

# 9. Job 6 — kube-state-metrics

```yaml
- job_name: 'kube-state-metrics'
```

This is another important component.

> kube-state-metrics tells Prometheus about the **state of Kubernetes objects**.

Examples:

```text
How many replicas should my Deployment have?

How many replicas are actually available?

Is a pod Running or Pending?

How many times did a container restart?

Is a Kubernetes Job completed?
```

Typical metrics include:

```text
kube_deployment_spec_replicas

kube_deployment_status_replicas_available

kube_pod_status_phase

kube_pod_container_status_restarts_total
```

Prometheus first discovers Kubernetes endpoints:

```yaml
kubernetes_sd_configs:
  - role: endpoints
```

and then keeps only the Service called:

```yaml
regex: kube-state-metrics
```

Conceptually:

```text
Prometheus
     |
     v
kube-state-metrics
     |
     v
Kubernetes API Server
     |
     v
Deployments / Pods / StatefulSets / Jobs
```

---

# 10. cAdvisor vs kube-state-metrics

This distinction is very important for new learners.

## cAdvisor

```text
cAdvisor
   ↓
Actual resource consumption

CPU = 40%

RAM = 500 MB
```

cAdvisor tells us:

> What resources are containers consuming?

---

## kube-state-metrics

```text
kube-state-metrics
   ↓
Kubernetes object state

Desired replicas = 3

Available replicas = 2

Pod phase = Pending

Restart count = 4
```

kube-state-metrics tells us:

> What does Kubernetes believe the state of its objects is?

---

# 11. Job 7 — Node Exporter

```yaml
- job_name: 'node'
```

Node Exporter collects Linux operating-system and hardware metrics from Kubernetes nodes.

Examples:

```text
CPU

Memory

Disk

Filesystem

Network

Load average
```

Typical metrics include:

```text
node_cpu_seconds_total

node_memory_MemAvailable_bytes

node_filesystem_avail_bytes

node_network_receive_bytes_total
```

The configuration discovers Kubernetes endpoints and keeps only:

```yaml
regex: node-exporter
```

Conceptually:

```text
Prometheus
     |
     v
Node Exporter
     |
     v
Linux Operating System
```

---

# 12. Why Do We Need Both Job 3 and Job 7?

A common beginner question is:

> **If Node Exporter already gives node CPU, memory, disk and network metrics, why do we also scrape kubelet metrics?**

The answer is:

> **They monitor different things.**

Job 3:

```yaml
job_name: kubernetes-nodes
```

scrapes **kubelet metrics**.

Job 7:

```yaml
job_name: node
```

scrapes **Node Exporter metrics**.

The easiest way to remember the difference is:

```text
Node Exporter
     ↓
How is the MACHINE doing?
```

while:

```text
Kubelet
     ↓
How is KUBERNETES managing this machine?
```

## Node Exporter

Node Exporter mainly gives operating-system and hardware metrics.

For example:

```text
CPU usage

Available memory

Disk usage

Filesystem capacity

Network traffic

System load
```

So if you want to know:

> "Is this server running out of RAM?"

or:

> "Is disk usage above 90%?"

Node Exporter is the main source.

---

## Kubelet Metrics

Kubelet is the Kubernetes agent running on every node.

Its metrics are more about Kubernetes operations on that node.

For example:

```text
Kubelet request metrics

Pod lifecycle operations

Container runtime operations

PLEG activity

Kubelet errors and latency

Kubelet resource-management behavior
```

So if the Linux server looks healthy but Kubernetes is having trouble starting, stopping, or managing pods, kubelet metrics can help explain why.

---

## Simple Comparison

| Job | Source | What it tells us |
|---|---|---|
| `kubernetes-nodes` | Kubelet | How Kubernetes is operating on the node |
| `node` | Node Exporter | How the underlying Linux machine is performing |

A good classroom explanation is:

> **Node Exporter tells us about the server. Kubelet metrics tell us about Kubernetes running on that server.**

---

## Is Node Exporter Alone Enough?

For basic infrastructure monitoring, **yes, it can be**.

If your requirement is only:

```text
CPU

Memory

Disk

Filesystem

Network
```

then Node Exporter may be sufficient.

But for proper Kubernetes monitoring, it is useful to keep both:

```text
Node Exporter
     ↓
Linux / machine health

Kubelet
     ↓
Kubernetes node-agent health
```

They complement each other rather than duplicate each other.

---


# 13. The Easiest Comparison for Learners

| Component | What does it tell us? |
|---|---|
| Prometheus | Health and performance of Prometheus itself |
| API Server | Kubernetes API Server metrics |
| Kubelet | Node and kubelet operational metrics |
| cAdvisor | Container CPU, memory, network and filesystem usage |
| Pod scraping | Application-specific metrics |
| kube-state-metrics | State of Kubernetes objects |
| Node Exporter | Linux operating-system and node metrics |

---

# 14. The Three Components Learners Often Confuse

A very useful way to explain them is:

```text
Node Exporter
     ↓
How is the MACHINE doing?

CPU / RAM / Disk / Network
```

```text
cAdvisor
     ↓
How are the CONTAINERS doing?

Container CPU / RAM / Network
```

```text
kube-state-metrics
     ↓
What does KUBERNETES think is happening?

Pod Running?

Deployment replicas?

Container restarts?

Job completed?
```

This distinction usually makes the overall monitoring architecture much easier to understand.

---

# 15. What is `relabel_configs`?

This is usually the most difficult-looking part of the Prometheus configuration for beginners.

You do not need to understand every regular expression at first.

Think of `relabel_configs` as:

> **A way to filter discovered targets and modify their labels or addresses before Prometheus scrapes them.**

The basic flow is:

```text
Discover
   ↓
Filter
   ↓
Modify
   ↓
Scrape
```

For example:

```yaml
- source_labels: [__meta_kubernetes_service_name]
  regex: node-exporter
  action: keep
```

This means:

```text
Discover all Services

nginx
grafana
mysql
redis
node-exporter
prometheus

       ↓

Keep only

node-exporter
```

---

# 16. What Does `labelmap` Do?

You may also see:

```yaml
- action: labelmap
  regex: __meta_kubernetes_pod_label_(.+)
```

This basically means:

> Copy useful Kubernetes labels into Prometheus labels.

For example, a pod might have:

```yaml
labels:
  app: payment
  environment: dev
```

These can become Prometheus labels such as:

```text
app="payment"

environment="dev"
```

Then Prometheus queries can filter metrics using those labels.

For example:

```promql
some_metric{
  app="payment",
  environment="dev"
}
```

---

# 17. Complete Monitoring Architecture

A simplified view of the entire configuration is:

```text
                         Kubernetes Cluster
                                |
                                |
                          +-------------+
                          | Prometheus  |
                          +-------------+
                                |
          +---------------------+-----------------------+
          |                     |                       |
          v                     v                       v
   API Server               Applications          Node Exporter
          |                  /metrics                  |
          |                                             |
          v                                             v
       Kubelet                                      Linux Node
          |
          +----------------+
          |                |
          v                v
      /metrics       /metrics/cadvisor
                            |
                            v
                       Containers


          Prometheus
               |
               v
       kube-state-metrics
               |
               v
       Kubernetes objects
       Pods / Deployments /
       StatefulSets / Jobs
```

---

# 18. The Four Monitoring Layers

Another simple way to teach the configuration is to divide it into four monitoring layers:

```text
Infrastructure
     ↓
Node Exporter
```

```text
Containers
     ↓
cAdvisor
```

```text
Kubernetes
     ↓
kube-state-metrics
Kubelet
API Server
```

```text
Applications
     ↓
Application /metrics endpoints
```

So the complete model becomes:

| Layer | Tool / Source |
|---|---|
| Infrastructure | Node Exporter |
| Containers | cAdvisor |
| Kubernetes | kube-state-metrics, Kubelet, API Server |
| Applications | Application `/metrics` endpoints |

---

# 19. Final Mental Model

The most important sentence for learners is:

> **Prometheus does not magically know what to monitor. `prometheus.yml` tells Prometheus how to discover targets, which targets to keep, and where to scrape metrics from.**

A good learning sequence is:

```text
1. Prometheus discovers something
          ↓
2. relabel_configs filters or modifies it
          ↓
3. Prometheus finds the metrics endpoint
          ↓
4. Prometheus calls /metrics
          ↓
5. Metrics are stored in the Prometheus time-series database
          ↓
6. PromQL can query those metrics
          ↓
7. Grafana can visualize them
```

In short:

```text
Targets
   ↓
Prometheus
   ↓
Metrics
   ↓
PromQL
   ↓
Grafana / Alerts
```

That is the core idea behind this entire configuration.
