# Conceptos base — de cero a autoscaling

> [🇺🇸 Read in English](concepts.md)

Este documento explica, en orden, los conceptos necesarios para entender el lab completo. Pensado para repasar antes de una entrevista o antes de explicarle el lab a alguien más.

## 1. Nodo vs Pod

- **Nodo** = una computadora física o virtual real. Tiene CPU y memoria RAM limitados.
- **Pod** = una copia de tu aplicación corriendo DENTRO de un Nodo. Un Nodo puede alojar varios Pods a la vez, mientras tenga CPU/RAM disponible.

```
Nodo (servidor real, ej: 8 CPU / 4GB RAM)
  ├── Pod 1 (copia de la app, usa 1 CPU / 500MB)
  ├── Pod 2 (copia de la app, usa 1 CPU / 500MB)
  └── Pod 3 (copia de la app, usa 1 CPU / 500MB)
```

## 2. Por qué un Pod queda en estado `Pending`

Kubernetes decide dónde poner un Pod comparando lo que el Pod **pide** (`resources.requests`) contra lo que el Nodo tiene **disponible**. Si ningún Nodo tiene suficiente CPU/memoria libre, el Pod se queda sin agendar — estado `Pending`.

```yaml
resources:
  requests:
    cpu: "500m"      # 500 milicores = medio core
    memory: "2Gi"    # 2 Gibibytes
  limits:
    cpu: "1"
    memory: "2Gi"
```

Comando para ver por qué un Pod está Pending:
```bash
kubectl describe pod <nombre> | tail -15
```

## 3. Taints y Tolerations

Un **taint** es una regla de exclusión en un Nodo: "no acepto Pods aquí, a menos que traigan permiso explícito". Una **toleration** es ese permiso, declarado en el Pod.

```yaml
# En el Nodo (taint)
taints:
  - key: "kwok-provider"
    value: "true"
    effect: "NoSchedule"

# En el Pod (toleration necesaria para poder correr ahí)
tolerations:
  - key: "kwok-provider"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

kwok pone un taint por defecto en sus nodos simulados, para que ningún Pod real caiga ahí "por accidente" — hay que darle permiso explícito.

## 4. HPA (Horizontal Pod Autoscaler)

El autoscaler de Pods nativo de Kubernetes — viene de fábrica, sin instalar nada. Solo sabe reaccionar a CPU/memoria.

```bash
kubectl get hpa
```

## 5. KEDA — HPA con esteroides

KEDA no reemplaza al HPA — genera un HPA automáticamente por debajo, pero le agrega la capacidad de vigilar +60 fuentes de eventos externos (colas de mensajes, tráfico HTTP, métricas custom), no solo CPU/memoria.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mi-scaler
spec:
  scaleTargetRef:
    name: mi-deployment
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "50"
```

## 6. Cluster Autoscaler — autoscaling de Nodos

Vigila si hay Pods en `Pending` por falta de recursos, y si los hay, le pide un Nodo nuevo al cloud provider configurado (AWS, GCP, Azure... o kwok, en este lab).

## 7. kwok — el simulador

No es un cluster alternativo a kind — es un **controller** que se instala DENTRO de un cluster real (kind, en este caso) y finge que ciertos Nodos existen, sin que haya hardware real detrás. Sirve para practicar autoscaling sin gastar en cloud real.

## 8. Karpenter — la alternativa moderna a Cluster Autoscaler

En vez de pedir nodos de "grupos predefinidos" (como hace Cluster Autoscaler), Karpenter le pide directamente al cloud provider "dame la instancia exacta que necesito ahorita", sin catálogos fijos. Resultado: escala en 45-60 segundos en vez de 3-5 minutos, y ahorra más dinero por bin-packing activo. Limitación: nativo de AWS (soporte en beta para Azure/GCP en 2026), mientras que Cluster Autoscaler funciona en cualquier cloud.

## 9. La cadena completa

```
KEDA vigila una métrica (ej: CPU)
    ↓
Decide cuántos Pods hacen falta → crea/borra Pods
    ↓
Si los Pods nuevos no caben en los Nodos existentes → quedan Pending
    ↓
Cluster Autoscaler (o Karpenter) detecta el Pending
    ↓
Pide un Nodo nuevo al cloud provider (o a kwok, en este lab)
    ↓
El Nodo nuevo aparece → los Pods se agendan ahí
```
