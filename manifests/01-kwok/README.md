# kwok — installation / instalación

---

## 🇺🇸 English

Unlike the other phases, kwok **required no custom YAML** — it was installed directly from the official project releases, via URL.

### Exact commands used

```bash
KWOK_REPO=kubernetes-sigs/kwok
KWOK_LATEST_RELEASE=v0.7.0   # check the actual latest version at:
                              # https://github.com/kubernetes-sigs/kwok/releases

# Installs the controller + CRDs + permissions (RBAC)
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/kwok.yaml"

# Installs the behavior rules — "fast" = simulated nodes become Ready almost instantly
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/stage-fast.yaml"
```

### What each file installs

| Remote file | What it brings |
|---|---|
| `kwok.yaml` | CRDs (`Stage`, `ClusterResourceStates`, etc.), the kwok-controller `Deployment`, ServiceAccount + RBAC |
| `stage-fast.yaml` | 5 `Stage` objects: `node-initialize`, `node-heartbeat-with-lease`, `pod-ready`, `pod-complete`, `pod-delete` — tells the controller how to behave at each stage of a simulated node/pod lifecycle |

### Verify

```bash
kubectl -n kube-system get pods -l app=kwok-controller
kubectl get stage
```

### Official reference

https://kwok.sigs.k8s.io/docs/user/kwok-in-cluster/

---

## 🇲🇽 Español

A diferencia de las otras fases, kwok **no requirió escribir ningún YAML propio** — se instaló directo desde los releases oficiales del proyecto, vía URL.

### Comandos exactos usados

```bash
KWOK_REPO=kubernetes-sigs/kwok
KWOK_LATEST_RELEASE=v0.7.0   # verificar la última versión real en:
                              # https://github.com/kubernetes-sigs/kwok/releases

# Instala el controller + CRDs + permisos (RBAC)
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/kwok.yaml"

# Instala las reglas de comportamiento — "fast" = nodos simulados pasan a Ready casi al instante
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/stage-fast.yaml"
```

### Qué instala cada uno (resumen)

| Archivo remoto | Qué trae |
|---|---|
| `kwok.yaml` | CRDs (`Stage`, `ClusterResourceStates`, etc.), el `Deployment` del kwok-controller, ServiceAccount + RBAC |
| `stage-fast.yaml` | 5 objetos `Stage`: `node-initialize`, `node-heartbeat-with-lease`, `pod-ready`, `pod-complete`, `pod-delete` — le dicen al controller cómo comportarse en cada etapa del ciclo de vida de un nodo/pod simulado |

### Verificación

```bash
kubectl -n kube-system get pods -l app=kwok-controller
kubectl get stage
```

### Referencia oficial

https://kwok.sigs.k8s.io/docs/user/kwok-in-cluster/
