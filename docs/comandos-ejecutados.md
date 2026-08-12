# Comandos ejecutados — log completo, fase por fase

> [🇺🇸 Read in English](commands-executed.md)

Prerrequisito: cluster de kind ya funcionando con `ingress-nginx` instalado.

```bash
kubectl config current-context   # confirmar: kind-<nombre-cluster>
```

---

## Fase 0 — App base, con réplicas FIJAS (escalado manual)

Punto de partida, antes de tocar cualquier autoscaler. Ver [`manifests/00-base-app/app.yaml`](../manifests/00-base-app/app.yaml).

```bash
kubectl apply -f manifests/00-base-app/app.yaml
kubectl get pods -l app=hello
kubectl get svc hello
kubectl get ingress hello
```

**Cómo "escalar" esto manualmente** (sin ningún autoscaler todavía):
```bash
kubectl scale deployment hello --replicas=5
kubectl get pods -l app=hello   # confirma que ahora hay 5, no 3
```

**El punto de este ejercicio:** entender que, sin autoscaling, el número de réplicas **nunca cambia solo** — si sube el tráfico a las 3am, alguien (tú) tiene que estar despierto para correr `kubectl scale` a mano. Todo lo que viene después (Fases 1-7) automatiza justo esta decisión.

---

## Fase 1 — Instalar kwok

```bash
KWOK_REPO=kubernetes-sigs/kwok
KWOK_LATEST_RELEASE=v0.7.0   # confirmar la última versión en:
                              # https://github.com/kubernetes-sigs/kwok/releases

kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/kwok.yaml"
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/stage-fast.yaml"
```

**Qué instala:**
- CRDs (`Stage`, `ClusterResourceStates`, etc.)
- El `kwok-controller` (Deployment en `kube-system`)
- Las reglas de comportamiento "fast" (nodos pasan a `Ready` casi instantáneo)

**Verificación:**
```bash
kubectl -n kube-system get pods -l app=kwok-controller
kubectl get stage
```

---

## Fase 2 — Cluster Autoscaler apuntando a kwok

```bash
helm repo add cluster-autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm upgrade --install cluster-autoscaler cluster-autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set "cloudProvider"=kwok \
  --set "autoDiscovery.clusterName"="kind-firstlab"
```

**Verificación:**
```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=kwok-cluster-autoscaler
kubectl -n kube-system get configmap kwok-provider-config kwok-provider-templates
```

⚠️ **Nota:** los templates de `kwok-provider-templates` son un snapshot de ejemplo de 2023 (Kubernetes v1.26), no reflejan el hardware real de tu máquina.

---

## Fase 3 — Probar Scale-UP de Nodos (manual)

Ver [`manifests/cluster-autoscaler/scale-test.yaml`](../manifests/cluster-autoscaler/scale-test.yaml).

```bash
kubectl apply -f manifests/cluster-autoscaler/scale-test.yaml
kubectl get pods -l app=scale-test -w
kubectl get nodes    # debe aparecer un nodo nuevo con VERSION: fake
```

⚠️ **Detalle encontrado:** el primer intento falló con `untolerated taint(s)` — kwok pone un taint `kwok-provider=true:NoSchedule` en sus nodos por defecto. Solución: agregar la `toleration` correspondiente al Pod (ver el YAML final en `manifests/`).

**Limpieza:**
```bash
kubectl delete -f manifests/cluster-autoscaler/scale-test.yaml
```

---

## Fase 4 — Instalar KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

**Verificación:**
```bash
kubectl get pods -n keda
```

---

## Fase 5 — Instalar Metrics Server (requisito del scaler de CPU)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

⚠️ **Detalle encontrado:** en kind, el pod queda `0/1 Running` por error de certificado TLS:
```
x509: cannot validate certificate for <IP> because it doesn't contain any IP SANs
```

**Solución (patch):**
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Verificación:**
```bash
kubectl top nodes
```

---

## Fase 6 — Scale-up/down de Pods con KEDA (scaler de CPU)

Ver [`manifests/keda/keda-demo.yaml`](../manifests/keda/keda-demo.yaml) y [`manifests/keda/keda-scaledobject.yaml`](../manifests/keda/keda-scaledobject.yaml).

```bash
kubectl apply -f manifests/keda/keda-demo.yaml
kubectl apply -f manifests/keda/keda-scaledobject.yaml

kubectl get scaledobject
kubectl get hpa
```

**Generar carga real de CPU:**
```bash
kubectl exec -it <pod-name> -- sh
# dentro del pod:
while true; do :; done
```

**Monitorear en otra terminal:**
```bash
while true; do clear; kubectl get pods,hpa -l app=keda-demo; sleep 2; done
```

**Resultado observado:** CPU llegó a `50%/50%` → escaló de 1 a 5 pods. Al detener la carga (`Ctrl+C` + `exit`), esperó el período de estabilización (~5 min) y redujo de vuelta a 1 pod.

---

## Fase 7 — Ciclo combinado: KEDA escala Pods → Cluster Autoscaler escala Nodos

Ajustes necesarios sobre la Fase 6 para forzar el desborde hasta Nodos:

1. Agregar `tolerations` (`kwok-provider=true`) al Deployment de KEDA — sin esto, aunque haya desborde, el CA no puede usar nodos kwok para acomodar los pods.
2. Subir `resources.requests.memory` del Deployment (ej: `1000Mi` por réplica) para que varias réplicas sí saturen la memoria del nodo real.
3. Subir `maxReplicaCount` en el `ScaledObject` para dar margen.

**Resultado observado (log real):**
```
CPU sube a 76-201% (sobre el umbral de 50%)
    → HPA/KEDA escala de 1 a 4 pods
    → 1 pod queda Pending (falta de memoria en el nodo real)
    → Cluster Autoscaler dispara TriggeredScaleUp
    → kwok crea nodo nuevo (VERSION: fake)
    → el pod se agenda ahí, pasa a Running
```

Esto confirma la cadena completa en un solo evento, sin necesidad de forzar el Scale-up de Nodos por separado.

---

## Housekeeping — comandos de limpieza frecuentes

```bash
kubectl get nodes                                  # ver nodos fantasma acumulados
kubectl delete node <nombre-nodo-fake>             # limpiar nodos fantasma viejos
kubectl delete deployment scale-test keda-demo     # limpiar deployments de prueba
```

⚠️ Los nodos fantasma de kwok **no desaparecen solos al reiniciar Docker Desktop** ni al apagar la Mac — hay que borrarlos manualmente cuando ya no se necesitan, o el Cluster Autoscaler los cuenta como "capacidad existente" en cálculos futuros.
