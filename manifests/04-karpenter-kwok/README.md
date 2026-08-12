# Extension: Karpenter + kwok / Extensión: Karpenter + kwok

---

## 🇺🇸 English

Replaces Cluster Autoscaler with Karpenter, using the same kwok simulation principle — but this time with Karpenter's **native kwok provider** (more modern than the generic Cluster Autoscaler chart).

Official source: https://github.com/kubernetes-sigs/karpenter/blob/main/kwok/README.md

### ⚠️ Important prerequisite

**Uninstall Cluster Autoscaler first** from Phase 2 — both can't coexist pointing at the same cluster:

```bash
helm uninstall cluster-autoscaler -n kube-system
```

### Additional tools required

- `make` (comes with Xcode Command Line Tools on Mac: `xcode-select --install`)
- `git` (to clone the repo)
- `go` (the Karpenter repo is written in Go — needed for compilation)

Verify you have them:
```bash
make --version
git --version
go version
```

If `go` is missing:
```bash
brew install go
```

### Step 1: Clone the Karpenter repo

```bash
git clone https://github.com/kubernetes-sigs/karpenter.git
cd karpenter/kwok
```

### Step 2: Environment variables (same pattern as regular kwok)

```bash
export KWOK_REPO=kind.local
export KIND_CLUSTER_NAME=firstlab   # adjust to your actual cluster name
```

### Step 3: Install kwok (Karpenter-specific version)

```bash
make install-kwok
```

### Step 4: Install and deploy Karpenter configured to use kwok

```bash
make apply
```

Repeat this command if you make code changes — it redeploys.

### Step 5: Create a NodePool (Karpenter's equivalent of a "node group")

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
      nodeClassRef:
        name: default
        kind: KWOKNodeClass
        group: karpenter.kwok.sh
  expireAfter: 720h
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 10s
---
apiVersion: karpenter.kwok.sh/v1alpha1
kind: KWOKNodeClass
metadata:
  name: default
EOF
```

### Step 6: Protect the real node (so Karpenter doesn't touch it)

```bash
kubectl taint nodes <your-real-node-name> CriticalAddonsOnly:NoSchedule
```

### Step 7: Reuse the same experiment from Phase 7

Apply `manifests/keda/keda-demo.yaml` and `keda-scaledobject.yaml` again — the flow should be identical, except now Karpenter (not Cluster Autoscaler) is the one creating fake nodes.

**Expected difference to observe:** labels on fake nodes will include Karpenter+kwok-exclusive fields, not available with Cluster Autoscaler+kwok:
```
karpenter.kwok.sh/instance-type
karpenter.kwok.sh/instance-size
karpenter.kwok.sh/instance-family
karpenter.kwok.sh/instance-cpu
karpenter.sh/instance-memory
```

Verify with:
```bash
kubectl get nodes --show-labels | grep karpenter
```

⚠️ These labels **only exist in the kwok provider** — they don't appear in a real Karpenter installation on AWS.

### Cleanup

```bash
make delete
make uninstall-kwok
```

### Comparison to document (for the final report)

| Aspect | Cluster Autoscaler + kwok | Karpenter + kwok |
|---|---|---|
| Scale-up speed (real, on AWS) | 3-5 min | 45-60 sec |
| Node model | Predefined node groups | Any instance, on demand |
| Installation | Simple Helm chart | Clone repo + `make` |
| Local simulation | Via generic chart | Via native kwok provider, more complete |
| Multi-cloud support | Yes (AWS, GCP, Azure, on-prem) | Mainly AWS; Azure/GCP in beta 2026 |

---

**Status of this section:** 🚧 pending execution and documentation with real results — next lab session.

---

## 🇲🇽 Español

Reemplaza al Cluster Autoscaler por Karpenter, usando el mismo principio de simulación con kwok — pero esta vez con el **proveedor kwok nativo de Karpenter** (más moderno que el chart genérico de Cluster Autoscaler).

Fuente oficial: https://github.com/kubernetes-sigs/karpenter/blob/main/kwok/README.md

### ⚠️ Prerrequisito importante

**Desinstala primero el Cluster Autoscaler** de la Fase 2 — no pueden convivir ambos apuntando al mismo cluster:

```bash
helm uninstall cluster-autoscaler -n kube-system
```

### Herramientas adicionales necesarias

- `make` (viene con Xcode Command Line Tools en Mac: `xcode-select --install`)
- `git` (para clonar el repo)
- `go` (el repo de Karpenter está escrito en Go — necesario para compilar)

Verifica que los tienes:
```bash
make --version
git --version
go version
```

Si falta `go`:
```bash
brew install go
```

### Paso 1: Clonar el repo de Karpenter

```bash
git clone https://github.com/kubernetes-sigs/karpenter.git
cd karpenter/kwok
```

### Paso 2: Variables de entorno (mismo patrón que kwok normal)

```bash
export KWOK_REPO=kind.local
export KIND_CLUSTER_NAME=firstlab   # ajustar al nombre real de tu cluster
```

### Paso 3: Instalar kwok (versión específica para Karpenter)

```bash
make install-kwok
```

### Paso 4: Instalar y desplegar Karpenter configurado para usar kwok

```bash
make apply
```

Repite este comando si haces cambios de código — vuelve a desplegar.

### Paso 5: Crear un NodePool (el equivalente en Karpenter a un "node group")

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
      nodeClassRef:
        name: default
        kind: KWOKNodeClass
        group: karpenter.kwok.sh
  expireAfter: 720h
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 10s
---
apiVersion: karpenter.kwok.sh/v1alpha1
kind: KWOKNodeClass
metadata:
  name: default
EOF
```

### Paso 6: Proteger el nodo real (para que Karpenter no lo toque)

```bash
kubectl taint nodes <nombre-de-tu-nodo-real> CriticalAddonsOnly:NoSchedule
```

### Paso 7: Reutilizar el mismo experimento de la Fase 7

Aplica de nuevo `manifests/keda/keda-demo.yaml` y `keda-scaledobject.yaml` — el flujo debería ser idéntico, solo que ahora Karpenter (no Cluster Autoscaler) es quien crea los nodos falsos.

**Diferencia esperada a observar:** las etiquetas en los nodos falsos van a incluir campos exclusivos de Karpenter+kwok, no disponibles con Cluster Autoscaler+kwok:
```
karpenter.kwok.sh/instance-type
karpenter.kwok.sh/instance-size
karpenter.kwok.sh/instance-family
karpenter.kwok.sh/instance-cpu
karpenter.sh/instance-memory
```

Verifica con:
```bash
kubectl get nodes --show-labels | grep karpenter
```

⚠️ Estas etiquetas **solo existen en el proveedor kwok** — no aparecen con una instalación real de Karpenter en AWS.

### Limpieza

```bash
make delete
make uninstall-kwok
```

### Comparación a documentar (para el reporte final)

| Aspecto | Cluster Autoscaler + kwok | Karpenter + kwok |
|---|---|---|
| Velocidad de scale-up (real, en AWS) | 3-5 min | 45-60 seg |
| Modelo de nodos | Node groups predefinidos | Cualquier instancia, a demanda |
| Instalación | Helm chart simple | Clonar repo + `make` |
| Simulación local | Vía chart genérico | Vía kwok provider nativo, más completo |
| Soporte multi-cloud | Sí (AWS, GCP, Azure, on-prem) | Principalmente AWS; Azure/GCP en beta 2026 |

---

**Estado de esta sección:** 🚧 pendiente de ejecutar y documentar con resultados reales — próxima sesión del lab.
