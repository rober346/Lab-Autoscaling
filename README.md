# Kubernetes Autoscaling Lab — kind + kwok + Cluster Autoscaler + KEDA (+ Karpenter)

> [🇲🇽 Leer en Español](README.es.md)

Hands-on Kubernetes autoscaling lab, fully simulated locally at zero cloud cost.

---

## 🎯 Purpose

Demonstrate the full Kubernetes autoscaling loop — from Pod-level scaling (KEDA) to Node-level scaling (Cluster Autoscaler / Karpenter) — without spending on real AWS/GCP infrastructure, using [kwok](https://kwok.sigs.k8s.io/) to simulate nodes.

---

## 🧩 Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    kind cluster (Docker, local)              │
│                                                              │
│   ┌──────────┐      ┌──────────────────┐      ┌────────────┐ │
│   │  KEDA    │─────▶│  Pods (replicas) │─────▶│  Pending?  │ │
│   │ (events) │      │   of the app     │      │            │ │
│   └──────────┘      └──────────────────┘      └─────┬──────┘ │
│                                                     │        │
│                                                     ▼        │
│                                        ┌──────────────────┐  │
│                                        │Cluster Autoscaler│  │
│                                        │ (or Karpenter)   │  │
│                                        └─────────┬────────┘  │
│                                                  │           │
│                                                  ▼           │
│                                        ┌──────────────────┐  │
│                                        │ kwok controller  │  │
│                                        │(simulated nodes) │  │
│                                        └──────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Node** = a real physical/virtual machine (server).
- **Pod** = a running copy of the app, living inside a Node.
- **KEDA** decides how many Pods are needed based on an external event (CPU, message queue, etc.).
- **Cluster Autoscaler / Karpenter** decides how many Nodes are needed to fit those Pods.
- **kwok** simulates those Nodes without real cloud cost.

---

## 📦 Stack

| Tool | Version tested | Role |
|---|---|---|
| [kind](https://kind.sigs.k8s.io/) | v0.36.1 (k8s v1.36.1) | Local Kubernetes cluster in Docker |
| [kwok](https://kwok.sigs.k8s.io/) | v0.7.0 | Fake-node simulator |
| [Cluster Autoscaler](https://github.com/kubernetes/autoscaler) | Helm chart, `cloudProvider=kwok` | Node autoscaling |
| [KEDA](https://keda.sh/) | Helm chart | Event-driven Pod autoscaling |
| [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) | latest | Real CPU/memory metrics |
| [Karpenter](https://karpenter.sh/) (WIP) | kwok provider | Modern alternative to Cluster Autoscaler |

---

## 📁 Repo structure

Numbered in chronological order — the sequence in which the lab was built, step by step:

```
.
├── README.md                          # this file (English)
├── README.es.md                       # Spanish version
├── docs/
│   ├── concepts.md                    # Node vs Pod, taints, HPA, etc — explained from scratch
│   ├── conceptos.md                   # Spanish version
│   ├── commands-executed.md           # full command log, phase by phase
│   ├── comandos-ejecutados.md         # Spanish version
│   ├── troubleshooting-en.md          # real errors encountered, and how they were solved
│   └── troubleshooting.md             # Spanish version
├── manifests/
│   ├── 00-base-app/                   # Deployment+Service+Ingress, FIXED replicas (manual)
│   ├── 01-kwok/                       # kwok installation references (node simulator)
│   ├── 02-cluster-autoscaler/         # Cluster Autoscaler + Node scale-up test
│   ├── 03-keda/                       # KEDA + automatic Pod scale-up/down test
│   └── 04-karpenter-kwok/             # extension: Karpenter simulated with kwok (WIP)
└── diagrams/                          # architecture diagrams
```

**Learning progression in one sentence:** first I learned to deploy an app with fixed replicas and real traffic (`00`), then to simulate infrastructure without spending money (`01`), then to automatically scale Nodes (`02`), then to automatically scale Pods (`03`), and now exploring the modern alternative to Cluster Autoscaler (`04`).

---

## 🚀 How to reproduce this lab

See [`docs/commands-executed.md`](docs/commands-executed.md) for the full step-by-step procedure, phase by phase, with explanations for every command.

---

## ✅ What was demonstrated

- [x] **Phase 0:** Real app deployed with **fixed** replicas (`replicas: 3`, 100% manual scaling) + Service + Ingress receiving traffic
- [x] **Phase 1:** kwok installed (controller + stages) inside a real kind cluster
- [x] **Phase 2:** Cluster Autoscaler configured with `cloudProvider=kwok`
- [x] **Phase 3:** Node scale-up triggered manually (Pod with excessive `requests.memory`)
- [x] **Phases 4-5:** KEDA + Metrics Server installed (with kind-specific TLS fix)
- [x] **Phase 6:** Pod scale-up/down with KEDA, **automatically**, based on real CPU
- [x] **Phase 7 (combined):** Full cycle in a single event — KEDA scales Pods → Pods don't fit → Cluster Autoscaler scales Nodes → kwok creates them
- [ ] **Phase 8 (WIP):** Same experiment, replacing Cluster Autoscaler with Karpenter (kwok provider)

**The key contrast this repo documents:** from Phase 0 (everything manual — you decide the replica count) to Phases 6-7 (everything automatic — the system decides based on real load).

---

## 👤 Author

Roberto Silveira

## 📄 License

MIT — free for educational use.
