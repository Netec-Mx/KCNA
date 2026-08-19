# 7 Práctica 5. Laboratorio integrador de fundamentos

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 75 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 5 – Repaso e Integración |

## Descripción General

Este laboratorio final integra todos los conceptos de los módulos 1 al 4. Trabajarás en tres fases: primero reconstruirás desde cero una arquitectura multi-namespace completa en un clúster de 3 nodos; después diagnosticarás 5 escenarios de fallo intencional que cubren errores comunes en contenedores, Pods, Deployments y Services; finalmente, responderás 10 preguntas tipo KCNA con autoevaluación. Todo el trabajo se documenta en `~/kcna-labs/lab05/`.

## Objetivos de Aprendizaje

- [ ] Integrar todos los conceptos del módulo desplegando una arquitectura completa desde cero en un clúster limpio de 3 nodos
- [ ] Diagnosticar y resolver 5 escenarios de fallo intencional utilizando `kubectl describe`, `logs` y `events`
- [ ] Demostrar fluidez en el uso de `kubectl` para administrar un entorno multi-namespace con múltiples tipos de recursos
- [ ] Responder preguntas tipo KCNA sobre arquitectura Kubernetes, objetos principales y scheduling con justificación técnica
- [ ] Leer e interpretar manifiestos YAML complejos identificando componentes, relaciones y posibles errores

## Prerrequisitos

### Conocimiento Previo

| Requisito | Descripción |
|-----------|-------------|
| Labs 01–04 completados | Dominio de Docker, Pods, Deployments, Services, ConfigMaps, Secrets, RBAC, PV/PVC |
| Arquitectura Kubernetes | Comprensión del plano de control (API Server, etcd, Scheduler, Controller Manager) y nodos de trabajo (kubelet, kube-proxy, container runtime) |
| Manifiestos YAML | Capacidad de leer y escribir manifiestos declarativos de Kubernetes |
| kubectl | Dominio de `apply`, `get`, `describe`, `logs`, `exec`, `delete` |

### Acceso Requerido

| Recurso | Detalle |
|---------|---------|
| Imagen en Docker Hub | `youruser/kcna-webapp:1.0.0` publicada y accesible (reemplaza `youruser` con tu usuario de Docker Hub) |
| Minikube 1.33.1 | Instalado y funcional con driver Docker |
| kubectl 1.30.2 | Configurado y apuntando al clúster Minikube |
| Conexión a Internet | Para descargar imágenes desde Docker Hub |

> **Nota:** A lo largo de este laboratorio, reemplaza `youruser` por tu nombre de usuario real de Docker Hub en todos los manifiestos y comandos.

## Entorno del Laboratorio

### Requisitos de Hardware

| Componente | Mínimo | Recomendado |
|------------|--------|-------------|
| CPU | 4 núcleos | 6+ núcleos |
| RAM | 8 GB | 16 GB |
| Disco | 30 GB libres (SSD) | 50 GB libres |

### Software Requerido

| Software | Versión |
|----------|---------|
| Docker Engine | 26.1.4 |
| Minikube | 1.33.1 |
| kubectl | 1.30.2 |
| Kubernetes (via Minikube) | 1.30.0 |
| jq | 1.6 |
| curl | 8.8.0 |

### Configuración Inicial del Entorno

```bash
# Crear directorio de trabajo
mkdir -p ~/kcna-labs/lab05
cd ~/kcna-labs/lab05

# Detener cualquier clúster Minikube existente
minikube delete --all

# Iniciar clúster limpio de 3 nodos
minikube start \
  --nodes=3 \
  --kubernetes-version=v1.30.0 \
  --driver=docker \
  --cpus=2 \
  --memory=2048

# Verificar que los 3 nodos están Ready
kubectl get nodes
```

**Salida esperada:**

```
NAME           STATUS   ROLES           AGE   VERSION
minikube       Ready    control-plane   60s   v1.30.0
minikube-m02   Ready    <none>          40s   v1.30.0
minikube-m03   Ready    <none>          20s   v1.30.0
```

```bash
# Verificar conectividad con el API Server
kubectl cluster-info

# Confirmar que la imagen está accesible
docker pull youruser/kcna-webapp:1.0.0
```

---

## Parte 1: Despliegue Completo desde Cero (30 minutos)

### Paso 1 — Crear los Namespaces

**Objetivo:** Establecer la estructura organizacional del clúster con dos namespaces aislados.

**Instrucciones:**

1. Crea el archivo de manifiesto para ambos namespaces:

```bash
cat > ~/kcna-labs/lab05/namespaces.yaml <<'ENDOFFILE'
apiVersion: v1
kind: Namespace
metadata:
  name: kcna-final
  labels:
    environment: production
    lab: "05"
---
apiVersion: v1
kind: Namespace
metadata:
  name: kcna-staging
  labels:
    environment: staging
    lab: "05"
ENDOFFILE
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab05/namespaces.yaml
```

**Salida esperada:**

```
namespace/kcna-final created
namespace/kcna-staging created
```

**Verificación:**

```bash
kubectl get namespaces --show-labels | grep kcna
```

Debes ver ambos namespaces con estado `Active` y las etiquetas correspondientes.

---

### Paso 2 — Crear ConfigMap y Secret en kcna-final

**Objetivo:** Configurar la aplicación con variables de entorno externalizadas y credenciales protegidas.

**Instrucciones:**

1. Crea el ConfigMap:

```bash
cat > ~/kcna-labs/lab05/configmap.yaml <<'ENDOFFILE'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: kcna-final
data:
  APP_ENV: "production"
  APP_VERSION: "1.0.0"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
ENDOFFILE
```

2. Crea el Secret:

```bash
cat > ~/kcna-labs/lab05/secret.yaml <<'ENDOFFILE'
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: kcna-final
type: Opaque
data:
  DB_PASSWORD: S2NuYUxhYjA1UGFzcw==
  API_KEY: YWJjZGVmZzEyMzQ1Njc4OQ==
ENDOFFILE
```

> **Nota:** Los valores están codificados en Base64. `S2NuYUxhYjA1UGFzcw==` decodifica a `KcnaLab05Pass` y `YWJjZGVmZzEyMzQ1Njc4OQ==` decodifica a `abcdefg123456789`.

3. Aplica ambos manifiestos:

```bash
kubectl apply -f ~/kcna-labs/lab05/configmap.yaml
kubectl apply -f ~/kcna-labs/lab05/secret.yaml
```

**Salida esperada:**

```
configmap/app-config created
secret/app-secret created
```

**Verificación:**

```bash
kubectl get configmap app-config -n kcna-final -o yaml
kubectl get secret app-secret -n kcna-final -o yaml
```

---

### Paso 3 — Desplegar kcna-webapp con 3 réplicas en kcna-final

**Objetivo:** Crear el Deployment principal con 3 réplicas, inyectando configuración desde ConfigMap y Secret.

**Instrucciones:**

1. Crea el manifiesto del Deployment:

```bash
cat > ~/kcna-labs/lab05/deployment-production.yaml <<'ENDOFFILE'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-final
  labels:
    app: kcna-webapp
    environment: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kcna-webapp
      environment: production
  template:
    metadata:
      labels:
        app: kcna-webapp
        environment: production
    spec:
      containers:
      - name: webapp
        image: youruser/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
          protocol: TCP
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "250m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
ENDOFFILE
```

> **Importante:** Antes de aplicar, edita el archivo y reemplaza `youruser` por tu usuario real de Docker Hub:
> ```bash
> sed -i 's/youruser/tu-usuario-dockerhub/g' ~/kcna-labs/lab05/deployment-production.yaml
> ```

2. Aplica el Deployment:

```bash
kubectl apply -f ~/kcna-labs/lab05/deployment-production.yaml
```

3. Espera a que las 3 réplicas estén listas:

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-final --timeout=120s
```

**Salida esperada:**

```
deployment "kcna-webapp" successfully rolled out
```

**Verificación:**

```bash
kubectl get deployment kcna-webapp -n kcna-final
kubectl get pods -n kcna-final -l app=kcna-webapp -o wide
```

Debes ver 3/3 réplicas disponibles y los Pods distribuidos en los nodos de trabajo.

---

### Paso 4 — Crear el Service NodePort

**Objetivo:** Exponer la aplicación externamente mediante un Service de tipo NodePort en el puerto 30080.

**Instrucciones:**

1. Crea el manifiesto del Service:

```bash
cat > ~/kcna-labs/lab05/service-production.yaml <<'ENDOFFILE'
apiVersion: v1
kind: Service
metadata:
  name: kcna-webapp-svc
  namespace: kcna-final
  labels:
    app: kcna-webapp
spec:
  type: NodePort
  selector:
    app: kcna-webapp
    environment: production
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
    protocol: TCP
ENDOFFILE
```

2. Aplica el Service:

```bash
kubectl apply -f ~/kcna-labs/lab05/service-production.yaml
```

**Salida esperada:**

```
service/kcna-webapp-svc created
```

**Verificación:**

```bash
# Verificar el Service
kubectl get svc kcna-webapp-svc -n kcna-final

# Verificar que los Endpoints están poblados
kubectl get endpoints kcna-webapp-svc -n kcna-final

# Probar conectividad
minikube service kcna-webapp-svc -n kcna-final --url
```

```bash
# Hacer una solicitud al endpoint
curl -s $(minikube service kcna-webapp-svc -n kcna-final --url) | jq .
```

**Salida esperada (ejemplo):**

```json
{
  "hostname": "kcna-webapp-7f8b9c6d4-abc12",
  "version": "1.0.0",
  "environment": "production"
}
```

---

### Paso 5 — Desplegar kcna-webapp-staging con nodeSelector

**Objetivo:** Crear un segundo Deployment en el namespace `kcna-staging` que se ejecute exclusivamente en un nodo específico usando `nodeSelector`.

**Instrucciones:**

1. Etiqueta el nodo `minikube-m02` para staging:

```bash
kubectl label node minikube-m02 environment=staging
```

2. Verifica la etiqueta:

```bash
kubectl get node minikube-m02 --show-labels | grep environment
```

3. Crea el ConfigMap y Secret para staging:

```bash
cat > ~/kcna-labs/lab05/configmap-staging.yaml <<'ENDOFFILE'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: kcna-staging
data:
  APP_ENV: "staging"
  APP_VERSION: "1.0.0"
  APP_PORT: "8080"
  LOG_LEVEL: "debug"
ENDOFFILE
```

```bash
cat > ~/kcna-labs/lab05/secret-staging.yaml <<'ENDOFFILE'
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: kcna-staging
type: Opaque
data:
  DB_PASSWORD: U3RhZ2luZ1Bhc3MxMjM=
  API_KEY: c3RhZ2luZ2tleS0wMDEK
ENDOFFILE
```

```bash
kubectl apply -f ~/kcna-labs/lab05/configmap-staging.yaml
kubectl apply -f ~/kcna-labs/lab05/secret-staging.yaml
```

4. Crea el Deployment de staging con `nodeSelector`:

```bash
cat > ~/kcna-labs/lab05/deployment-staging.yaml <<'ENDOFFILE'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp-staging
  namespace: kcna-staging
  labels:
    app: kcna-webapp
    environment: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kcna-webapp
      environment: staging
  template:
    metadata:
      labels:
        app: kcna-webapp
        environment: staging
    spec:
      nodeSelector:
        environment: staging
      containers:
      - name: webapp
        image: youruser/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
ENDOFFILE
```

> **Importante:** Antes de aplicar, edita el archivo y reemplaza `youruser` por tu usuario real de Docker Hub:
> ```bash
> sed -i 's/youruser/tu-usuario-dockerhub/g' ~/kcna-labs/lab05/deployment-staging.yaml
> ```

5. Aplica el Deployment:

```bash
kubectl apply -f ~/kcna-labs/lab05/deployment-staging.yaml
```

6. Espera al rollout:

```bash
kubectl rollout status deployment/kcna-webapp-staging -n kcna-staging --timeout=120s
```

**Verificación:**

```bash
# Confirmar que todos los Pods están en minikube-m02
kubectl get pods -n kcna-staging -o wide
```

Todos los Pods de staging deben mostrar `minikube-m02` en la columna NODE.

---
