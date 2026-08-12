# Kubernetes Autoscaling Lab — kind + kwok + Cluster Autoscaler + KEDA (+ Karpenter)

> [🇺🇸 Read in English](README.md)

Lab práctico de autoscaling en Kubernetes, simulado localmente sin costo de cloud real.

---

## 🎯 Objetivo

Demostrar el ciclo completo de autoscaling en Kubernetes — desde el escalamiento de Pods (KEDA) hasta el escalamiento de Nodos (Cluster Autoscaler / Karpenter) — sin gastar en infraestructura real de AWS/GCP, usando [kwok](https://kwok.sigs.k8s.io/) para simular nodos.

---

## 🧩 Arquitectura

```
┌──────────────────────────────────────────────────────────────────┐
│                     kind cluster (Docker, local)                 │
│                                                                  │
│   ┌──────────┐      ┌──────────────────┐      ┌────────────┐     │
│   │  KEDA    │─────▶│  Pods (réplicas) │─────▶│  Pending?  │     │
│   │ (eventos)│      │   de la app      │      │            │     │
│   └──────────┘      └──────────────────┘      └─────┬──────┘     │
│                                                     │            │
│                                                     ▼            │
│                                            ┌──────────────────┐  │
│                                            │Cluster Autoscaler│  │
│                                            │ (o Karpenter)    │  │
│                                            └─────────┬────────┘  │
│                                                      │           │
│                                                      ▼           │
│                                           ┌──────────────────┐   │
│                                           │ kwok controller  │   │
│                                           │(nodos simulados) │   │
│                                           └──────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

**Conceptos clave:**
- **Nodo** = una computadora física o virtual real (servidor).
- **Pod** = una copia de la app corriendo dentro de un Nodo.
- **KEDA** decide cuántos Pods hacen falta según un evento externo (CPU, cola de mensajes, etc.).
- **Cluster Autoscaler / Karpenter** decide cuántos Nodos hacen falta para que quepan esos Pods.
- **kwok** simula esos Nodos sin gastar en cloud real.

---

## 📦 Stack usado

| Herramienta | Versión probada | Rol |
|---|---|---|
| [kind](https://kind.sigs.k8s.io/) | v0.36.1 (k8s v1.36.1) | Cluster de Kubernetes local en Docker |
| [kwok](https://kwok.sigs.k8s.io/) | v0.7.0 | Simulador de nodos falsos |
| [Cluster Autoscaler](https://github.com/kubernetes/autoscaler) | Helm chart, `cloudProvider=kwok` | Autoscaling de Nodos |
| [KEDA](https://keda.sh/) | Helm chart | Autoscaling de Pods basado en eventos |
| [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) | latest | Métricas reales de CPU/memoria |
| [Karpenter](https://karpenter.sh/) (en progreso) | kwok provider | Alternativa moderna a Cluster Autoscaler |

---

## 📁 Estructura del repo

Numerada en orden cronológico — así se fue construyendo el aprendizaje, paso a paso:

```
.
├── README.md                          # versión en inglés / English version
├── README.es.md                       # este archivo
├── docs/
│   ├── conceptos.md                   # Node vs Pod, taints, HPA, etc — explicado desde cero
│   ├── concepts.md                    # versión en inglés
│   ├── comandos-ejecutados.md         # log completo de comandos, fase por fase
│   ├── commands-executed.md           # versión en inglés
│   ├── troubleshooting.md             # errores reales que salieron, y cómo se resolvieron
│   └── troubleshooting-en.md          # versión en inglés
├── manifests/
│   ├── 00-base-app/                   # Deployment+Service+Ingress, réplicas FIJAS (manual)
│   ├── 01-kwok/                       # referencias de instalación de kwok (simulador de nodos)
│   ├── 02-cluster-autoscaler/         # Cluster Autoscaler + prueba de scale-up de Nodos
│   ├── 03-keda/                       # KEDA + prueba de scale-up/down de Pods (automático)
│   └── 04-karpenter-kwok/             # extensión: Karpenter simulado con kwok (en progreso)
└── diagrams/                          # diagramas de arquitectura
```

**Progresión:** Desplegar una app con réplicas fijas y tráfico real (`00`), luego simular infraestructura sin gastar dinero (`01`), luego a escalar Nodos automáticamente (`02`), luego a escalar Pods automáticamente (`03`), Explorar una alternativa moderna a Cluster Autoscaler (`04`).

---

## 🚀 Cómo reproducir este lab

Ver [`docs/comandos-ejecutados.md`](docs/comandos-ejecutados.md) para el procedimiento completo, fase por fase, con explicación de cada comando.

---

## ✅ Qué se demostró

- [x] **Fase 0:** App real desplegada con réplicas **fijas** (`replicas: 3`, escalado 100% manual) + Service + Ingress recibiendo tráfico
- [x] **Fase 1:** Instalación de kwok (controller + stages) dentro de un cluster kind real
- [x] **Fase 2:** Cluster Autoscaler configurado con `cloudProvider=kwok`
- [x] **Fase 3:** Scale-up de Nodos forzado manualmente (Pod con `requests.memory` excesivo)
- [x] **Fases 4-5:** Instalación de KEDA + Metrics Server (con fix de TLS específico para kind)
- [x] **Fase 6:** Scale-up/down de Pods con KEDA, **automático**, basado en CPU real
- [x] **Fase 7 (combinada):** Ciclo completo en un solo evento — KEDA escala Pods → Pods no caben → Cluster Autoscaler escala Nodos → kwok los crea
- [ ] **Fase 8 (en progreso):** Mismo experimento, reemplazando Cluster Autoscaler por Karpenter (kwok provider)

**El contraste clave que documenta este repo:** de la Fase 0 (todo manual, se decide el número de réplicas) a la Fase 6-7 (todo automático, el sistema decide solo según carga real).

---

## 👤 Autor

Roberto Silveira

## 📄 Licencia

MIT — libre uso educativo.
