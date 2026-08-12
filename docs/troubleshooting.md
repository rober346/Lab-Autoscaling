# Troubleshooting — errores reales encontrados y su solución

> [🇺🇸 Read in English](troubleshooting-en.md)

Todos estos errores ocurrieron durante el desarrollo real de este lab (no son hipotéticos) — se documentan para referencia futura y para demostrar el proceso de debugging, no solo el resultado final.

---

## 1. `0/1 nodes are available: 1 Insufficient memory... untolerated taint(s)`

**Contexto:** al forzar un Pod con `requests.memory` alto para probar el Cluster Autoscaler + kwok.

**Causa:** kwok pone un taint `kwok-provider=true:NoSchedule` en los nodos que crea, para evitar que Pods reales caigan ahí "por accidente". El Pod de prueba no traía la `toleration` correspondiente.

**Solución:**
```yaml
tolerations:
  - key: "kwok-provider"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

---

## 2. `x509: cannot validate certificate for 172.18.0.2 because it doesn't contain any IP SANs`

**Contexto:** Metrics Server instalado en kind, pod quedaba `0/1 Running` (no crash, pero tampoco `Ready`).

**Causa:** Metrics Server, por defecto, valida certificados TLS estrictos entre nodos — kind no configura certificados con esas IPs incluidas (SANs).

**Solución:**
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

⚠️ Esta bandera es aceptable en un cluster local de práctica, **nunca en producción real** sin evaluar el riesgo.

---

## 3. `dial tcp <IP>:10250: connect: no route to host` (en logs de Metrics Server)

**Contexto:** después de crear nodos falsos con kwok, Metrics Server intentaba conectarse a ellos para pedir métricas reales.

**Causa:** los nodos de kwok no tienen ningún proceso real corriendo en el puerto del kubelet — es esperado que Metrics Server no pueda leerlos. No bloquea el resto del sistema, solo genera ruido en logs.

**Solución:** ninguna necesaria — es comportamiento esperado. Opcionalmente, borrar nodos fantasma que ya no se usan para reducir el ruido:
```bash
kubectl delete node <nombre-nodo-fake>
```

---

## 4. Pod nuevo se sigue agendando en un nodo fantasma ya borrado

**Contexto:** al borrar un nodo fake (`kubectl delete node`), un Pod que ya estaba corriendo ahí no se reasigna solo al nodo real.

**Causa:** Kubernetes no migra Pods automáticamente cuando el nodo donde viven desaparece por debajo — el Pod queda "huérfano" hasta que algo fuerza su recreación.

**Solución:**
```bash
kubectl delete pod <nombre-del-pod>   # el ReplicaSet lo recrea automáticamente
```

---

## 5. `kubectl exec` falla con `no route to host` al intentar entrar a un Pod

**Contexto:** al intentar generar carga de CPU dentro de un Pod para probar KEDA.

**Causa:** el Pod había quedado agendado en un nodo fantasma de kwok (sin proceso real detrás) — no en el nodo real.

**Solución:** confirmar con `kubectl get pods -o wide` en qué nodo cayó el Pod. Si es un nodo fake, borrar el pod y el nodo fake, y dejar que se recree en el nodo real.

---

## 6. `helm install` falla con `cluster unreachable`

**Contexto:** después de reiniciar la Mac, al intentar instalar un chart de Helm.

**Causa:** Docker Desktop tardó unos segundos más en terminar de levantar el contenedor de kind — el API server todavía no respondía.

**Solución:** esperar unos segundos y reintentar el mismo comando. Verificar con:
```bash
docker ps | grep <nombre-cluster>
kubectl cluster-info
```

---

## 7. Deployment de prueba viejo (`scale-test`) seguía corriendo sin querer

**Contexto:** un `kubectl delete -f archivo.yaml` no se había completado correctamente en una sesión anterior, y el Deployment siguió vivo, consumiendo memoria del nodo real y "contaminando" pruebas posteriores.

**Causa:** posible error de red momentáneo al aplicar el delete, sin confirmación explícita del resultado en su momento.

**Solución:** verificar siempre con `kubectl get deployment,pods` después de cualquier `delete`, no asumir que se completó solo porque el comando no mostró error inmediato. Borrar por nombre directo si hay duda:
```bash
kubectl delete deployment <nombre>
```
