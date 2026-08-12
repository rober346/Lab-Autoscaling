# Troubleshooting — real errors encountered and how they were solved

> [🇲🇽 Leer en Español](troubleshooting.md)

All of these errors occurred during the actual development of this lab (not hypothetical) — documented for future reference and to show the debugging process, not just the final result.

---

## 1. `0/1 nodes are available: 1 Insufficient memory... untolerated taint(s)`

**Context:** while forcing a Pod with high `requests.memory` to test Cluster Autoscaler + kwok.

**Cause:** kwok places a `kwok-provider=true:NoSchedule` taint on the nodes it creates, to prevent real Pods from landing there "by accident". The test Pod didn't carry the corresponding `toleration`.

**Fix:**
```yaml
tolerations:
  - key: "kwok-provider"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

---

## 2. `x509: cannot validate certificate for 172.18.0.2 because it doesn't contain any IP SANs`

**Context:** Metrics Server installed in kind, pod stayed at `0/1 Running` (not crashing, but never `Ready`).

**Cause:** Metrics Server, by default, enforces strict TLS certificate validation between nodes — kind does not configure certificates with those IPs included (SANs).

**Fix:**
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

⚠️ This flag is acceptable on a local practice cluster, **never in real production** without evaluating the risk.

---

## 3. `dial tcp <IP>:10250: connect: no route to host` (in Metrics Server logs)

**Context:** after creating fake nodes with kwok, Metrics Server tried to connect to them to fetch real metrics.

**Cause:** kwok nodes have no real process running on the kubelet port — it's expected that Metrics Server can't read them. This doesn't block the rest of the system, just generates log noise.

**Fix:** none needed — this is expected behavior. Optionally, delete ghost nodes that are no longer in use to reduce noise:
```bash
kubectl delete node <fake-node-name>
```

---

## 4. New Pod keeps getting scheduled on an already-deleted ghost node

**Context:** after deleting a fake node (`kubectl delete node`), a Pod that was already running there didn't get rescheduled automatically to the real node.

**Cause:** Kubernetes does not migrate Pods automatically when the node they live on disappears underneath them — the Pod becomes "orphaned" until something forces its recreation.

**Fix:**
```bash
kubectl delete pod <pod-name>   # the ReplicaSet recreates it automatically
```

---

## 5. `kubectl exec` fails with `no route to host` when entering a Pod

**Context:** while trying to generate CPU load inside a Pod to test KEDA.

**Cause:** the Pod had landed on a kwok ghost node (no real process behind it) — not on the real node.

**Fix:** confirm with `kubectl get pods -o wide` which node the Pod landed on. If it's a fake node, delete the pod and the fake node, and let it recreate on the real node.

---

## 6. `helm install` fails with `cluster unreachable`

**Context:** after restarting the Mac, trying to install a Helm chart.

**Cause:** Docker Desktop took a few extra seconds to finish bringing up the kind container — the API server wasn't responding yet.

**Fix:** wait a few seconds and retry the same command. Verify with:
```bash
docker ps | grep <cluster-name>
kubectl cluster-info
```

---

## 7. Old test Deployment (`scale-test`) kept running unintentionally

**Context:** a `kubectl delete -f file.yaml` hadn't completed correctly in a previous session, and the Deployment kept running, consuming memory from the real node and "contaminating" subsequent tests.

**Cause:** possible momentary network error when applying the delete, without explicit confirmation of the result at the time.

**Fix:** always verify with `kubectl get deployment,pods` after any `delete`, don't assume it completed just because the command didn't show an immediate error. Delete by name directly if in doubt:
```bash
kubectl delete deployment <name>
```
