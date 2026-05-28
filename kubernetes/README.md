# 🚀 Clúster de Alta Disponibilidad — Laravel + MySQL sobre Máquinas Virtuales (HomeLab Cloud)

Este proyecto documenta cómo construí un entorno altamente disponible utilizando **Laravel**, **MySQL** y **Kubernetes** sobre mis propias máquinas virtuales locales, simulando la arquitectura de un proveedor cloud como AWS o Google Cloud.

El objetivo fue diseñar un **Home Lab Bare-Metal** capaz de tolerar fallos físicos reales, incluyendo:

* Alta disponibilidad de aplicaciones.
* Almacenamiento distribuido.
* Failover automático/manual.
* Seguridad desacoplada de Git.
* Red virtual avanzada.
* Persistencia resiliente.

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

# 📊 Monitorización del Clúster

# 4️⃣ Kubernetes Dashboard

Desplegué el Dashboard oficial para:

* Supervisar Pods.
* Ver consumo de CPU/RAM.
* Monitorizar estado de nodos.
* Validar failovers.
* Gestionar workloads visualmente.

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

Aunque las VMs tenían recursos disponibles.

---

## 🔍 Causa

Existía un `LimitRange` olvidado en el namespace.

Cada contenedor reservaba automáticamente:

```txt
500m CPU
```

Como Laravel usa Sidecar:

* PHP-FPM
* Nginx

La reserva total era:

```txt
1000m CPU
```

Kubernetes calculaba además:

* Calico
* Longhorn
* System Pods

Y bloqueaba el scheduling preventivamente.

---

## ✅ Solución

Eliminé el `LimitRange` y definí requests reales:

```yaml
resources:
  requests:
    cpu: "150m"
```

Resultado:

* MySQL volvió a desplegarse inmediatamente.
* Mejor aprovechamiento del clúster.
* Failover funcional.

---

# ⚠️ Problema #2 — Bloqueo del Volumen (Split-Brain)

## 🐛 Síntoma

Al apagar violentamente la VM de almacenamiento:

* El PVC quedó congelado ~5 minutos.
* MySQL dejó de responder temporalmente.

---

## 🔍 Causa

Longhorn protege automáticamente contra:

# Split-Brain

Evita escritura simultánea sobre réplicas inconsistentes.

---

## ✅ Solución

Forcé el desalojo del nodo y recreación inmediata:

```bash
kubectl drain slave1-ubuntu-kubernetes \
  --force \
  --ignore-daemonsets \
  --delete-emptydir-data
```

```bash
kubectl delete pod mysql-0 \
  --force \
  --grace-period=0
```

Resultado:

* Failover < 30 segundos.
* Sin pérdida de datos.
* Recuperación automática al volver el nodo.

---

# 📸 Evidencias del Clúster

# ✅ Infraestructura Estable

## Kubernetes Dashboard

Visualización del estado saludable de nodos y workloads:

---

## Longhorn UI — Estado Healthy

Volumen de MySQL correctamente replicado:

---

## Laravel funcionando

Aplicación desplegada mediante Ingress Controller:

---

# ⚠️ Comportamiento ante Fallos

## 🔄 Migración Automática del Pod MySQL

Failover del StatefulSet al apagar un nodo:

---

## 🟡 Longhorn en modo Degraded

El sistema mantiene disponibilidad incluso tras la caída de una VM:

---

## ♻️ Recuperación Automática

Resincronización automática tras volver el nodo:

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

Todo ejecutándose sobre hardware local y máquinas virtuales propias.