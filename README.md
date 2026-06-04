# 🚀 Clúster de Alta Disponibilidad — Laravel + MySQL sobre Máquinas Virtuales (HomeLab Cloud)

Este proyecto documenta cómo construí un entorno altamente disponible utilizando **Laravel**, **MySQL** y **Kubernetes** sobre mis propias máquinas virtuales locales, simulando la arquitectura de un proveedor cloud como AWS o Google Cloud.

El objetivo fue diseñar un **Home Lab Bare-Metal** capaz de tolerar fallos físicos reales, incluyendo:

* Alta disponibilidad de aplicaciones.
* Almacenamiento distribuido.
* Failover automático/manual.
* Seguridad desacoplada de Git.
* Red virtual avanzada.
* Persistencia resiliente.
* Observabilidad y monitorización del clúster.

---

# 🏗️ Arquitectura General

## 📦 Stack Principal

* Kubernetes
* Laravel
* MySQL
* Nginx
* PHP-FPM (Patrón Sidecar)
* Longhorn
* Calico (Tigera Operator)
* Nginx Ingress Controller
* Kubernetes Dashboard
* Metrics Server

---

# 🔌 Infraestructura de Red, Seguridad y Almacenamiento

# 1️⃣ Red con Calico + DNS Casero

## 📡 CNI: Calico mediante Tigera Operator

Instalé Calico aplicando los manifiestos oficiales:

```bash
kubectl apply -f tigera-operator.yaml
kubectl apply -f custom-resources.yaml
```

Posteriormente edité el recurso personalizado para definir manualmente el CIDR del clúster:

```yaml
cidr: 192.168.0.0/16
```

Esto permitió:

* Coordinar las IPs de Pods y nodos.
* Mantener consistencia de red.
* Garantizar replicación distribuida estable.

---

## 🌐 DNS Local mediante `/etc/hosts`

Para evitar depender de DNS públicos configuré dominios locales apuntando directamente al Ingress del clúster:

```txt
192.168.X.X laravel.app.com
192.168.X.X longhorn.local
```

---

# 🔐 Seguridad Avanzada

# 2️⃣ ConfigMaps y Secrets desacoplados de Git

Separé completamente:

* Configuración pública.
* Credenciales privadas.

De esta forma ninguna credencial sensible queda almacenada en GitHub.

---

## 📦 ConfigMap de Aplicación

Variables públicas compartidas:

```yaml
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
APP_URL=http://laravel.app.com
```

---

## 🔑 Secret de Base de Datos

Generado directamente desde CLI:

```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD='tu_password_secreto'
```

---

## 🐳 Secret para GitHub Container Registry (GHCR)

Como las imágenes Docker de Laravel son privadas:

```bash
kubectl create secret docker-registry github-registry-secret \
  --docker-server=ghcr.io \
  --docker-username='user' \
  --docker-password='PAT' \
  --docker-email='email'
```

Esto permite que Kubernetes descargue imágenes privadas de forma segura.

---

## 🔒 Basic Auth para Longhorn

Protección de la interfaz web:

```bash
htpasswd -c auth admin
```

```bash
kubectl create secret generic longhorn-auth-secret \
  --from-file=auth \
  -n longhorn-system
```

---

# 💾 Almacenamiento Distribuido con Longhorn

# 3️⃣ Persistencia tipo Cloud (EBS-like)

Implementé Longhorn para simular discos distribuidos similares a AWS EBS.

Cuando MySQL solicita un PVC:

```yaml
storageClassName: longhorn
```

Longhorn:

* Crea automáticamente el volumen.
* Replica datos entre nodos.
* Mantiene sincronización en tiempo real.
* Tolera caída de nodos físicos.

---

# 📊 Monitorización y Observabilidad

# 4️⃣ Kubernetes Dashboard

Desplegué el Dashboard oficial para:

* Supervisar Pods.
* Ver consumo de CPU/RAM.
* Monitorizar estado de nodos.
* Validar failovers.
* Gestionar workloads visualmente.

---

# 5️⃣ Metrics Server

Para obtener métricas en tiempo real instalé Metrics Server:

```bash
kubectl apply -f metrics-server.yaml
```

Una vez desplegado fue posible consultar el consumo real de recursos del clúster:

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

Metrics Server fue clave para:

* Analizar consumo real de CPU y memoria.
* Diagnosticar problemas de scheduling.
* Detectar configuraciones incorrectas de recursos.
* Validar el comportamiento de los Pods durante failovers.
* Correlacionar eventos del Scheduler con el uso efectivo del clúster.

---

# 🧠 Post-Mortem Técnico

# ⚠️ Problema #1 — `Insufficient CPU`

## 🐛 Síntoma

Al apagar `slave1`, MySQL quedaba en estado:

```txt
Pending
```

Mostrando:

```txt
Insufficient cpu
```

Aunque las VMs parecían tener recursos disponibles.

---

## 🔍 Investigación

Utilizando Metrics Server pude comprobar el consumo real del clúster:

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

Las métricas indicaban que el uso real de CPU era relativamente bajo, por lo que el problema no estaba relacionado con falta de capacidad física.

---

## 🔍 Causa

Tras revisar la configuración descubrí que existía un `LimitRange` heredado de pruebas anteriores.

Cada contenedor reservaba automáticamente:

```txt
500m CPU
```

Como Laravel utiliza un patrón Sidecar:

* PHP-FPM
* Nginx

La reserva efectiva por Pod era:

```txt
1000m CPU
```

Kubernetes calculaba además los recursos reservados por:

* Calico
* Longhorn
* Metrics Server
* Dashboard
* System Pods

Aunque el consumo real era bajo, el Scheduler bloqueaba la reubicación del Pod tras la caída de un nodo porque las reservas comprometidas superaban la capacidad disponible.

---

## ✅ Solución

Eliminé el `LimitRange` y definí requests acordes al consumo real observado:

```yaml
resources:
  requests:
    cpu: "150m"
```

Resultado:

* MySQL volvió a desplegarse inmediatamente.
* Mejor aprovechamiento de recursos.
* Failover funcional.
* Requests ajustados a métricas reales.

---

# ⚠️ Problema #2 — Bloqueo del Volumen (Split-Brain)

## 🐛 Síntoma

Al apagar violentamente la VM de almacenamiento:

* El PVC quedó congelado durante varios minutos.
* MySQL dejó de responder temporalmente.
* El volumen no se liberaba del nodo apagado

---

## 🔍 Causa

Longhorn protege automáticamente contra escenarios de:

# Split-Brain

Longhorn evita que múltiples réplicas inconsistentes acepten escrituras simultáneamente tras una pérdida abrupta de conectividad. Cuando un nodo se apaga de forma inesperada, el volumen queda bloqueado temporalmente porque Kubernetes no puede determinar si la caída es permanente o si el nodo volverá a estar disponible en pocos minutos. Este mecanismo evita posibles corrupciones de datos derivadas de escrituras concurrentes.
---

## ✅ Solución

Indique al nodo como fuera de servicio y recreo el pod en otro nodo satisfactoriamente:

```bash`
kubectl taint nodes slave3-ubuntu-kubernetes node.kubernetes.io/out-of-service=nodeshutdown:NoExecute
```

Resultado:

* Failover inferior a 30 segundos.
* Sin pérdida de datos.
* Recuperación automática al reincorporar el nodo.

---

# 📸 Evidencias del Clúster

# ✅ Infraestructura Estable

## Kubernetes Dashboard

Visualización del estado saludable de nodos y workloads.

<img src="img/dashboard_kubernetes.png" alt="dashboard kubernetes" width="100%">
<img src="img/dashboard_workloads1.png" alt="dashboard workloads" width="100%">
<img src="img/dashboard_workloads2.png" alt="dashboard workloads" width="100%">

---

## Longhorn UI — Estado Healthy

Volumen de MySQL correctamente replicado entre nodos.

<img src="img/dashboard_longhorn.png" alt="dashboard longhorn" width="100%">
<img src="img/dashboard_longhorn_volume_nodess.png" alt="dashboard volume nodes" width="100%">
<img src="img/dashboard_longhorn_nodes.png" alt="dashboard longhorn nodes" width="100%">

---

## Laravel funcionando

Aplicación desplegada mediante Ingress Controller.

<img src="img/laravel_app.png" alt="laravel app" width="100%">

---

# ⚠️ Comportamiento ante Fallos

## 🔄 Migración Automática del Pod MySQL

Failover del StatefulSet tras apagar un nodo.

<img src="img/pods-before-taint.png" alt="pods_before_tainted" width="100%">
<img src="img/pods-after-tainted-node.png" alt="pods_after_tainted" width="100%">

---

## 🟡 Longhorn en modo Degraded

El sistema mantiene disponibilidad incluso durante la caída de una VM.

<img src="longhorn_degraded.png" alt="longhorn_degraded" width="100%">
---

## ♻️ Recuperación Automática

<img src="img/pods_sync.png" alt="pods_sync" width="100%">

---

## ♻️ Recuperación del volumen al nodo una vez asignado como untainted 

<img src="img/rebulding_volumen_after_untainted.png" alt="pods_sync" width="100%">

---

# 🚀 RoadMap

## 🔄 Automatización CI/CD

Implementar:

* GitHub Actions
* GitLab CI

Objetivos:

* Build automático.
* Push a registry privado.
* Deploy automático en Kubernetes.

---

## 🧪 Estrategia Multi-Entorno

Separar:

* `staging`
* `production`

Utilizando Namespaces.

---

## ⚙️ GitOps

Integrar:

* ArgoCD

Para gestión declarativa completa de infraestructura y aplicaciones.

---

# 🏁 Conclusión

Este Home Lab me permitió replicar conceptos reales de infraestructura cloud:

* Alta disponibilidad.
* Persistencia distribuida.
* Orquestación avanzada.
* Seguridad desacoplada.
* Failover resiliente.
* Redes internas complejas.
* Observabilidad mediante Metrics Server y Kubernetes Dashboard.
* Diagnóstico y resolución de problemas reales de operación.

Todo ejecutándose sobre hardware local y máquinas virtuales propias, reproduciendo patrones y desafíos habituales en entornos productivos basados en Kubernetes.
