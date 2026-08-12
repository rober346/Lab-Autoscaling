# Commands executed — full log, phase by phase

> [🇲🇽 Leer en Español](comandos-ejecutados.md)

Prerequisite: a kind cluster already running with `ingress-nginx` installed.

```bash
kubectl config current-context   # confirm: kind-<cluster-name>
```

---

## Phase 0 — Base app, with FIXED replicas (manual scaling)

Starting point, before touching any autoscaler. See [`manifests/00-base-app/app.yaml`](../manifests/00-base-app/app.yaml).

```bash
kubectl apply -f manifests/00-base-app/app.yaml
kubectl get pods -l app=hello
kubectl get svc hello
kubectl get ingress hello
```

**How to "scale" this manually** (no autoscaler yet):
```bash
kubectl scale deployment hello --replicas=5
kubectl get pods -l app=hello   # confirms there are now 5, not 3
```

**The point of this exercise:** understand that without autoscaling, the replica count **never changes on its own** — if traffic spikes at 3am, someone (you) has to be awake to run `kubectl scale` by hand. Everything that follows (Phases 1-7) automates exactly this decision.

---

## Phase 1 — Install kwok

```bash
KWOK_REPO=kubernetes-sigs/kwok
KWOK_LATEST_RELEASE=v0.7.0   # confirm the latest version at:
                              # https://github.com/kubernetes-sigs/kwok/releases

kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/kwok.yaml"
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/stage-fast.yaml"
```

**What gets installed:**
- CRDs (`Stage`, `ClusterResourceStates`, etc.)
- The `kwok-controller` (Deployment in `kube-system`)
- "Fast" behavior rules (simulated nodes transition to `Ready` almost instantly)

**Verify:**
```bash
kubectl -n kube-system get pods -l app=kwok-controller
kubectl get stage
```

---

## Phase 2 — Cluster Autoscaler pointing at kwok

```bash
helm repo add cluster-autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm upgrade --install cluster-autoscaler cluster-autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set "cloudProvider"=kwok \
  --set "autoDiscovery.clusterName"="kind-firstlab"
```

**Verify:**
```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=kwok-cluster-autoscaler
kubectl -n kube-system get configmap kwok-provider-config kwok-provider-templates
```

⚠️ **Note:** the `kwok-provider-templates` ConfigMap is a 2023 snapshot (Kubernetes v1.26) and does not reflect your machine's actual hardware.

---

## Phase 3 — Test Node Scale-UP (manual trigger)

See [`manifests/cluster-autoscaler/scale-test.yaml`](../manifests/cluster-autoscaler/scale-test.yaml).

```bash
kubectl apply -f manifests/cluster-autoscaler/scale-test.yaml
kubectl get pods -l app=scale-test -w
kubectl get nodes    # a new node should appear with VERSION: fake
```

⚠️ **Detail found:** the first attempt failed with `untolerated taint(s)` — kwok places a `kwok-provider=true:NoSchedule` taint on its nodes by default. Fix: add the corresponding `toleration` to the Pod (see the final YAML in `manifests/`).

**Cleanup:**
```bash
kubectl delete -f manifests/cluster-autoscaler/scale-test.yaml
```

---

## Phase 4 — Install KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

**Verify:**
```bash
kubectl get pods -n keda
```

---

## Phase 5 — Install Metrics Server (required for CPU scaler)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

⚠️ **Detail found:** in kind, the pod stays at `0/1 Running` due to a TLS certificate error:
```
x509: cannot validate certificate for <IP> because it doesn't contain any IP SANs
```

**Fix (patch):**
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Verify:**
```bash
kubectl top nodes
```

---

## Phase 6 — Pod scale-up/down with KEDA (CPU scaler)

See [`manifests/keda/keda-demo.yaml`](../manifests/keda/keda-demo.yaml) and [`manifests/keda/keda-scaledobject.yaml`](../manifests/keda/keda-scaledobject.yaml).

```bash
kubectl apply -f manifests/keda/keda-demo.yaml
kubectl apply -f manifests/keda/keda-scaledobject.yaml

kubectl get scaledobject
kubectl get hpa
```

**Generate real CPU load:**
```bash
kubectl exec -it <pod-name> -- sh
# inside the pod:
while true; do :; done
```

**Monitor in another terminal:**
```bash
while true; do clear; kubectl get pods,hpa -l app=keda-demo; sleep 2; done
```

**Observed result:** CPU reached `50%/50%` → scaled from 1 to 5 pods. After stopping the load (`Ctrl+C` + `exit`), waited through the cooldown period (~5 min) and scaled back down to 1 pod.

---

## Phase 7 — Combined cycle: KEDA scales Pods → Cluster Autoscaler scales Nodes

Adjustments needed on top of Phase 6 to force the overflow all the way to Nodes:

1. Add `tolerations` (`kwok-provider=true`) to the KEDA Deployment — without this, even when there's overflow, the CA can't use kwok nodes to schedule the pods.
2. Raise `resources.requests.memory` on the Deployment (e.g.: `1000Mi` per replica) so that multiple replicas saturate the real node's memory.
3. Raise `maxReplicaCount` in the `ScaledObject` to give room.

**Observed result (real log):**
```
CPU spikes to 76-201% (above the 50% threshold)
    → HPA/KEDA scales from 1 to 4 pods
    → 1 pod stays Pending (not enough memory on the real node)
    → Cluster Autoscaler fires TriggeredScaleUp
    → kwok creates a new node (VERSION: fake)
    → the pod gets scheduled there, transitions to Running
```

This confirms the full chain in a single event, without needing to manually trigger the Node scale-up separately.

---

## Housekeeping — frequent cleanup commands

```bash
kubectl get nodes                                  # see accumulated ghost nodes
kubectl delete node <fake-node-name>               # clean up old ghost nodes
kubectl delete deployment scale-test keda-demo     # clean up test deployments
```

⚠️ kwok's ghost nodes **do not disappear on their own when Docker Desktop restarts** or when you shut down your Mac — they must be deleted manually when no longer needed, or the Cluster Autoscaler will count them as "existing capacity" in future calculations.
