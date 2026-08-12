# Core concepts — from zero to autoscaling

> [🇲🇽 Leer en Español](conceptos.md)

This document explains, in order, the concepts needed to understand the full lab. Good for reviewing before an interview or before walking someone through the lab.

## 1. Node vs Pod

- **Node** = a real physical or virtual machine. It has a fixed amount of CPU and RAM.
- **Pod** = a running copy of your application, living INSIDE a Node. A single Node can host multiple Pods at the same time, as long as it has enough available CPU/RAM.

```
Node (real server, e.g.: 8 CPU / 4 GB RAM)
  ├── Pod 1 (copy of the app, uses 1 CPU / 500 MB)
  ├── Pod 2 (copy of the app, uses 1 CPU / 500 MB)
  └── Pod 3 (copy of the app, uses 1 CPU / 500 MB)
```

## 2. Why a Pod ends up in `Pending` state

Kubernetes decides where to place a Pod by comparing what the Pod **requests** (`resources.requests`) against what each Node has **available**. If no Node has enough free CPU/memory, the Pod remains unscheduled — `Pending` state.

```yaml
resources:
  requests:
    cpu: "500m"      # 500 millicores = half a core
    memory: "2Gi"    # 2 Gibibytes
  limits:
    cpu: "1"
    memory: "2Gi"
```

Command to see why a Pod is Pending:
```bash
kubectl describe pod <name> | tail -15
```

## 3. Taints and Tolerations

A **taint** is an exclusion rule on a Node: "I don't accept Pods here, unless they carry explicit permission". A **toleration** is that permission, declared in the Pod spec.

```yaml
# On the Node (taint)
taints:
  - key: "kwok-provider"
    value: "true"
    effect: "NoSchedule"

# On the Pod (toleration required to run there)
tolerations:
  - key: "kwok-provider"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

kwok places a default taint on its simulated nodes so that no real Pod lands there "by accident" — you have to grant explicit permission.

## 4. HPA (Horizontal Pod Autoscaler)

Kubernetes' built-in Pod autoscaler — ships out of the box, no installation needed. Can only react to CPU/memory metrics.

```bash
kubectl get hpa
```

## 5. KEDA — HPA on steroids

KEDA doesn't replace the HPA — it generates one automatically under the hood, but adds the ability to watch 60+ external event sources (message queues, HTTP traffic, custom metrics), not just CPU/memory.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-scaler
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "50"
```

## 6. Cluster Autoscaler — Node autoscaling

Watches for Pods stuck in `Pending` due to resource shortages, and when found, requests a new Node from the configured cloud provider (AWS, GCP, Azure… or kwok, in this lab).

## 7. kwok — the simulator

Not an alternative cluster to kind — it's a **controller** installed INSIDE a real cluster (kind, in this case) that pretends certain Nodes exist, without any real hardware behind them. Useful for practicing autoscaling without spending on real cloud infrastructure.

## 8. Karpenter — the modern alternative to Cluster Autoscaler

Instead of requesting nodes from "predefined groups" (as Cluster Autoscaler does), Karpenter asks the cloud provider directly: "give me the exact instance I need right now", without fixed catalogs. Result: scales in 45-60 seconds instead of 3-5 minutes, and saves more money through active bin-packing. Limitation: natively AWS-only (Azure/GCP support in beta as of 2026), while Cluster Autoscaler works on any cloud.

## 9. The full chain

```
KEDA watches a metric (e.g.: CPU)
    ↓
Decides how many Pods are needed → creates/deletes Pods
    ↓
If new Pods don't fit in existing Nodes → they stay Pending
    ↓
Cluster Autoscaler (or Karpenter) detects the Pending Pod
    ↓
Requests a new Node from the cloud provider (or from kwok, in this lab)
    ↓
New Node appears → Pods get scheduled there
```
