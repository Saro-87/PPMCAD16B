# Elastic Stack – Centralized Logging for VMs and Kubernetes

## 1. The Problem: Troubleshooting Logs Across Multiple Servers

Suppose an application is running across **10 VMs / servers**.

A user reports an intermittent issue:

> “Sometimes I am getting logged out of the application.”

The issue is assigned to the DevOps/Operations team for investigation.

Without centralized logging, the troubleshooting process may look like this:

```text
User reports logout issue
        |
        v
DevOps team starts investigation
        |
        v
SSH into Server 1
Check application.log
Check access.log
Check error.log
        |
        v
SSH into Server 2
Check the same logs
        |
        v
...
        |
        v
SSH into Server 10
```

Typical commands might include:

```bash
ssh user@server-ip

cd /var/log/application

tail -f application.log
grep ERROR application.log
grep "user123" application.log
```

The team may need to inspect several files:

```text
application.log
access.log
error.log
authentication.log
audit.log
```

This approach becomes difficult because:

* Logs are distributed across many servers.
* The engineer has to identify which server handled the user's request.
* The exact time of the problem may not be known.
* Multiple log files may need to be correlated.
* Developers may also require server access.
* Intermittent problems can be difficult to reproduce.
* Searching several GBs of logs manually can be slow.

The fundamental problem is:

> **The logs are generated in many places, but there is no single place from which we can search and analyze them.**

---

# 2. Centralized Logging

A better approach is to collect logs from all servers and send them to a **central logging platform**.

Conceptually:

```text
Server 1 ──┐
Server 2 ──┤
Server 3 ──┤
Server 4 ──┤
Server 5 ──┤
           ├──> Central Log Platform
Server 6 ──┤
Server 7 ──┤
Server 8 ──┤
Server 9 ──┤
Server 10 ─┘
```

Now DevOps engineers and developers can investigate the issue from one location instead of logging into every server.

This is one of the primary use cases of the **Elastic Stack**.

---

# 3. Elastic Stack

The Elastic Stack is commonly used for centralized log collection, storage, search, analysis, and visualization.

Traditionally, it was often referred to as the **ELK Stack**:

```text
E = Elasticsearch
L = Logstash
K = Kibana
```

Basic architecture:

```text
Application Servers
        |
        v
     Logstash
        |
        v
  Elasticsearch
        |
        v
      Kibana
```

Each component has a different responsibility.

---

# 4. Elasticsearch

Elasticsearch is the central component responsible for storing and searching the collected data.

For logging, think of Elasticsearch as the place where logs from all your servers become centrally searchable.

Instead of:

```text
Server 1 -> application.log
Server 2 -> application.log
Server 3 -> application.log
...
Server 10 -> application.log
```

we can have:

```text
Server 1 ──┐
Server 2 ──┤
Server 3 ──┤
           ├──> Elasticsearch
...        │
Server 10 ─┘
```

A log might originally look like:

```text
2026-08-22 10:15:32 ERROR User session expired for user 12345
```

After processing, it can become structured data such as:

```json
{
  "@timestamp": "2026-08-22T10:15:32",
  "level": "ERROR",
  "user_id": "12345",
  "message": "User session expired",
  "server": "app-server-07"
}
```

Structured logs make searching and filtering much easier.

For example, we may search for:

```text
user_id = 12345
```

or:

```text
level = ERROR
```

or:

```text
server = app-server-07
```

or combine several conditions.

---

# 5. Logstash

Logstash is a data processing pipeline.

It can:

1. Read data from different sources.
2. Parse and transform the data.
3. Send the processed data somewhere else.

A useful mental model is:

```text
INPUT
  |
  v
FILTER
  |
  v
OUTPUT
```

In Logstash terminology:

```text
Input -> Filter -> Output
```

For our application servers:

```text
application.log ──┐
access.log ───────┤
error.log ────────┤
                  v
               Logstash
                  |
           Parse / Transform
                  |
                  v
            Elasticsearch
```

A simplified Logstash configuration might look like:

```text
input {
    file {
        path => "/var/log/myapp/*.log"
    }
}

filter {
    # Parse or transform log data
}

output {
    elasticsearch {
        hosts => ["http://elasticsearch:9200"]
    }
}
```

The exact configuration depends on the application's log format.

---

# 6. Kibana

Kibana provides the user interface for interacting with data stored in Elasticsearch.

Architecture:

```text
                     Elasticsearch
                          |
                          v
                       Kibana
                          |
             +------------+------------+
             |            |            |
             v            v            v
          Search      Dashboard    Visualization
```

Using Kibana, DevOps engineers and developers can:

* Search logs.
* Filter logs.
* Investigate errors.
* Build dashboards.
* Analyze events over time.
* Correlate application activity across servers.

For example, instead of SSHing into ten servers, we could search centrally for:

```text
user.id : "12345"
```

and select:

```text
Last 30 minutes
```

Kibana could then show events from all servers that processed requests for that user.

---

# 7. Investigating the Logout Problem with Elastic Stack

Returning to our original incident:

> A user is intermittently getting logged out.

Without centralized logging:

```text
SSH -> Server 1
SSH -> Server 2
SSH -> Server 3
...
SSH -> Server 10
```

With centralized logging:

```text
               Kibana
                  |
          Search user ID
                  |
                  v
             Elasticsearch
                  |
      +-----------+-----------+
      |           |           |
   Server 2    Server 5    Server 8
    logs         logs         logs
```

Suppose the engineer searches for:

```text
user.id = 12345
```

and finds:

```text
10:14:52 LOGIN successful
10:15:10 Request /products
10:15:21 Request /cart
10:15:32 SESSION_EXPIRED
10:15:33 User redirected to /login
```

The logs also show:

```text
server = app-server-07
```

Now the engineer has an investigation path.

For example, they may discover:

```text
app-server-07
      |
      v
Session validation failed
      |
      v
Session data missing from shared session store
```

The issue may then require:

* infrastructure remediation,
* configuration changes,
* application changes,
* or involvement from the development team.

The key point is that centralized logging makes the investigation much easier.

---

# 8. Why Running Logstash Everywhere Can Become Expensive

Logstash is a Java-based application.

Because it runs on the JVM and can perform significant processing, it can consume noticeable:

```text
CPU
Memory
```

Imagine having:

```text
10 application servers
```

and running a relatively heavy Logstash process on every server:

```text
Server 1  -> Application + Logstash
Server 2  -> Application + Logstash
Server 3  -> Application + Logstash
...
Server 10 -> Application + Logstash
```

The application and the logging agent are now competing for resources.

For simple log shipping, running a complete Logstash instance on every machine may therefore be unnecessary.

---

# 9. Beats

Elastic introduced lightweight data shippers known as **Beats**.

Different Beats were designed for different types of data.

Examples include:

```text
Filebeat      -> Log files
Metricbeat    -> System/application metrics
Heartbeat     -> Uptime / availability checks
Auditbeat     -> Audit-related events
```

For centralized application logging, **Filebeat** is especially relevant.

---

# 10. Filebeat

Filebeat is a lightweight agent that monitors log files and forwards log events.

Example:

```text
Server
 |
 |-- application
 |
 |-- application.log
 |-- access.log
 |-- error.log
 |
 `-- Filebeat
       |
       v
   Elasticsearch
```

For ten servers:

```text
Server 1  -> Filebeat ──┐
Server 2  -> Filebeat ──┤
Server 3  -> Filebeat ──┤
                        |
...                     ├──> Elasticsearch
                        |
Server 10 -> Filebeat ──┘
```

Kibana then queries Elasticsearch:

```text
Filebeat
   |
   v
Elasticsearch
   |
   v
Kibana
```

---

# 11. Filebeat vs Logstash

A simple distinction is:

| Component | Primary Role                                     |
| --------- | ------------------------------------------------ |
| Filebeat  | Lightweight log collection and forwarding        |
| Logstash  | More powerful data processing and transformation |


---

# 12. Is Logstash Always Required?

No.

A simpler architecture can be:

```text
Filebeat
   |
   v
Elasticsearch
   |
   v
Kibana
```

If extensive transformation is required, Logstash may be introduced:

```text
Filebeat
   |
   v
Logstash
   |
   v
Elasticsearch
   |
   v
Kibana
```

So we should not think of Elastic logging as always requiring:

```text
Filebeat + Logstash + Elasticsearch + Kibana
```

The architecture depends on the requirement.

---

# 13. Logging in Kubernetes

Now consider the same problem in Kubernetes.

Suppose we have:

```text
Kubernetes Cluster
        |
        +-- Worker Node 1
        +-- Worker Node 2
        +-- Worker Node 3
        ...
        +-- Worker Node 10
```

An e-commerce application is running on this cluster.

There may be many pods:

```text
frontend pods
backend pods
payment pods
cart pods
authentication pods
order pods
inventory pods
```

For example:

```text
Worker Node 1
  |
  +-- frontend-7fd89
  +-- cart-56bd7
  +-- auth-76cd8

Worker Node 2
  |
  +-- frontend-98abcd
  +-- payment-54abc
  +-- order-88ac9
```

---

# 14. kubectl logs

For an individual pod we can inspect logs using:

```bash
kubectl logs <pod-name> -n <namespace>
```

Example:

```bash
kubectl logs auth-service-7f5c8d8b9d-abc12 -n ecommerce
```

To continuously follow logs:

```bash
kubectl logs -f auth-service-7f5c8d8b9d-abc12 -n ecommerce
```

This is useful for immediate troubleshooting.

However, imagine hundreds of pods.

Searching each pod individually quickly becomes difficult.

---

# 15. Pods Are Dynamic

Kubernetes makes centralized logging even more important because pods are not permanent machines.

Pods can:

```text
start
stop
restart
fail
move to another node
scale up
scale down
```

For example:

```text
auth-pod-1
    |
    X  Pod crashes
```

Kubernetes may create:

```text
auth-pod-2
```

If our troubleshooting strategy depends only on connecting to the currently running pod, historical logs may become difficult to investigate.

A centralized logging platform allows the logs to survive independently of the lifecycle of the pod.

---

# 16. How Kubernetes Container Logs Reach the Node

Applications generally write logs to:

```text
stdout
stderr
```

Example:

```text
Application Container
        |
        +--> stdout
        |
        +--> stderr
```

The container runtime makes these logs available on the Kubernetes node.

Conceptually:

```text
Pod
 |
 v
Container
 |
 v
stdout / stderr
 |
 v
Node-level container log files
```

A logging agent running on that node can then collect them.

---

# 17. DaemonSet

In Kubernetes, log collection agents are commonly deployed as a **DaemonSet**.

A DaemonSet generally ensures that a copy of the logging pod runs on every eligible worker node.

For example:

```text
Worker Node 1
   |
   +-- Application Pods
   +-- Filebeat Pod

Worker Node 2
   |
   +-- Application Pods
   +-- Filebeat Pod

Worker Node 3
   |
   +-- Application Pods
   +-- Filebeat Pod
```

For ten nodes:

```text
10 Worker Nodes
      |
      v
Approximately one logging-agent pod per eligible node
```

This is useful because each agent can collect logs generated by containers running on its own node.

---

# 18. Kubernetes Centralized Logging Architecture

A typical architecture could look like:

```text
                   Kubernetes Cluster

        +-------------------------------------+
        |                                     |
        | Worker Node 1                       |
        |   Application Pods                  |
        |   Filebeat / Fluent Bit             |
        |                                     |
        | Worker Node 2                       |
        |   Application Pods                  |
        |   Filebeat / Fluent Bit             |
        |                                     |
        | Worker Node 3                       |
        |   Application Pods                  |
        |   Filebeat / Fluent Bit             |
        |                                     |
        +------------------+------------------+
                           |
                           v
                     Elasticsearch
                           |
                           v
                         Kibana
```

Now the logs from pods across all nodes can be searched centrally.

---

# 19. EFK Stack

In Kubernetes environments, you will frequently hear the term:

```text
EFK Stack
```

Traditionally:

```text
E = Elasticsearch
F = Fluentd / Filebeat
K = Kibana
```

In modern environments, **Fluent Bit** is also commonly used as the lightweight log collector.

So an architecture may look like:

```text
Fluent Bit
    |
    v
Elasticsearch
    |
    v
Kibana
```

or:

```text
Filebeat
    |
    v
Elasticsearch
    |
    v
Kibana
```

---


# 20. Complete Kubernetes Logging Flow

A useful end-to-end mental model is:

```text
User
 |
 v
Application
 |
 v
Container
 |
 +--> stdout / stderr
 |
 v
Kubernetes Node
 |
 v
Container Log Files
 |
 v
Fluent Bit / Filebeat
 |
 v
Elasticsearch
 |
 v
Kibana
 |
 v
DevOps / Developers
```

So when a customer reports:

> “I was logged out at approximately 10:15 AM.”

the engineer can search centrally using information such as:

```text
Time: 10:10 - 10:20

user.id = 12345
```

and potentially correlate logs across several services:

```text
Frontend
    |
    v
Authentication Service
    |
    v
Session Service
    |
    v
Redis
```

That is significantly easier than manually running:

```bash
kubectl logs
```

against many individual pods.

---

# Elasticsearch Ingest Pipelines

## What is an Ingest Pipeline?

An **Elasticsearch ingest pipeline** processes a log event **before Elasticsearch indexes and stores it**.

Example raw log:

```text
2026-08-16T15:10:00Z app[1234] ERROR Database connection timeout
```

Filebeat may collect this as one `message` field. The ingest pipeline can parse it into structured fields such as:

```text
@timestamp
app.pid
log.level
app.message
```

This allows precise searches in Kibana such as:

```text
log.level: "ERROR"
```

instead of searching for `ERROR` inside the entire message.

---

## Flow

```text
Pod stdout
    |
    v
Container log
    |
    v
Filebeat
    |
    | Adds Kubernetes metadata
    v
Elasticsearch Ingest Pipeline
    |
    | Parse / Transform / Normalize
    v
Elasticsearch Index
    |
    v
Kibana
```

In the lab:

```text
plain logs -> plain-app-logs pipeline -> Grok parsing

JSON logs  -> json-app-logs pipeline  -> JSON parsing
```

Filebeat uses the Kubernetes `log_format` label to select the appropriate pipeline.

---

## Does It Replace Logstash?

**For many simple log-processing requirements, yes.**

Traditional architecture:

```text
Filebeat
   |
   v
Logstash
   |
   v
Elasticsearch
```

With ingest pipelines:

```text
Filebeat
   |
   v
Elasticsearch Ingest Pipeline
   |
   v
Elasticsearch
```

Therefore, **Logstash is not required in this lab**.

---

### Easy way to remember

```text
Filebeat       -> Collect logs

Logstash       -> Advanced processing and routing

Ingest Pipeline -> Parse/transform inside Elasticsearch

Elasticsearch  -> Store and search

Kibana         -> Search and visualize
```

> **Ingest pipelines can replace Logstash when the main requirement is parsing and transforming logs before indexing.**

---
