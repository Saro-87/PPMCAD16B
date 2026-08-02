# End-to-end auto-scaling in Kubernetes

## Background

Kubernetes has two logical layers:

- **Control plane** — the "brain"; manages scheduling, state, and decisions
- **Worker nodes** — where actual application pods run

---

## The scenario

Your application is running on K8s and suddenly sees a surge in load. These are legitimate user requests, so you need to scale up to handle them, not drop them.

---

## Answer 1 - Scale the pods (HPA)

Your application runs as a `Deployment`. You attach an **HPA (Horizontal Pod Autoscaler)** to it and configure a policy:

> "If CPU or memory on any pod exceeds 80%, scale up."

The HPA continuously monitors resource metrics and adds replicas automatically:

- Starts at **3 replicas**
- Scales to **8**, then **16** (the configured max)

At 16 replicas, you notice **6 pods stuck in `Pending` state**, they can't be scheduled because none of the existing worker nodes have enough CPU or memory left.

---

## Answer 2 — Scale the nodes (Cluster Autoscaler)

The **Cluster Autoscaler** is a background process that watches the cluster for pending pods. When it finds them, it checks *why* they're pending.

If the reason is insufficient resources (no CPU/memory available), it sends a scale-out request to the cloud provider's **Auto Scaling Group**, which spins up a new worker node:

```
2 nodes → 3 nodes → 4 nodes  (as needed)
```

Once the new node joins the cluster, the scheduler places the pending pods on it, and all user requests start getting fulfilled.

---