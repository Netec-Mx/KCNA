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

### Paso 6 — Exponer staging y validar la arquitectura completa

**Objetivo:** Exponer el Deployment de staging con un Service ClusterIP, validar la conectividad interna y confirmar que la arquitectura multi-namespace completa (producción + staging) está operativa. Este paso cierra la Parte 1.

**Instrucciones:**

1. Crea un Service ClusterIP para staging:

```bash
cat > ~/kcna-labs/lab05/service-staging.yaml <<'ENDOFFILE'
apiVersion: v1
kind: Service
metadata:
  name: kcna-webapp-staging-svc
  namespace: kcna-staging
  labels:
    app: kcna-webapp
    environment: staging
spec:
  type: ClusterIP
  selector:
    app: kcna-webapp
    environment: staging
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
ENDOFFILE

kubectl apply -f ~/kcna-labs/lab05/service-staging.yaml
```

2. Valida la conectividad interna al Service de staging desde un Pod temporal:

```bash
kubectl run tmp-check --rm -it --restart=Never \
  --image=busybox:1.36.1 -n kcna-staging -- \
  wget -qO- http://kcna-webapp-staging-svc.kcna-staging.svc.cluster.local:8080/health
```

3. Genera un mapa de la arquitectura completa desplegada:

```bash
echo "=== Namespace kcna-final (producción) ==="
kubectl get deploy,svc,cm,secret -n kcna-final
echo ""
echo "=== Namespace kcna-staging ==="
kubectl get deploy,svc,cm,secret -n kcna-staging
```

**Salida esperada:**

```
{"status":"healthy"}
```

Y el mapa muestra en `kcna-final` el Deployment `kcna-webapp` (3/3) con su NodePort, y en `kcna-staging` el Deployment `kcna-webapp-staging` (2/2) con su ClusterIP, cada uno con su ConfigMap y Secret.

**Verificación:**

```bash
PROD=$(kubectl get deploy kcna-webapp -n kcna-final -o jsonpath='{.status.readyReplicas}')
STG=$(kubectl get deploy kcna-webapp-staging -n kcna-staging -o jsonpath='{.status.readyReplicas}')
echo "Producción: $PROD/3 | Staging: $STG/2"
[ "$PROD" = "3" ] && [ "$STG" = "2" ] && echo "✅ Arquitectura completa operativa" || echo "❌ Revisa los Deployments"
```

> **Cierre de la Parte 1:** Has reconstruido desde cero una arquitectura cloud native completa con dos entornos aislados por namespace, configuración externalizada (ConfigMap/Secret), despliegues con réplicas y probes, exposición con Services (NodePort y ClusterIP) y ubicación dirigida con `nodeSelector`. En la Parte 2 la someterás a fallos.

---

## Parte 2: Diagnóstico de 5 Escenarios de Fallo (25 minutos)

En esta fase inyectarás **cinco fallos intencionales** que representan los errores más comunes de examen sobre contenedores, Pods, Deployments y Services. Para cada uno aplicarás la metodología: **Observar → Describir → Inspeccionar → Corregir → Verificar**.

### Paso 7 — Inyectar los escenarios de fallo

**Objetivo:** Crear los recursos deliberadamente rotos en un namespace dedicado y sobre el entorno de producción.

**Instrucciones:**

1. Crea el namespace de escenarios y aplica los recursos rotos:

```bash
kubectl create namespace kcna-fallos

cat > ~/kcna-labs/lab05/fallos.yaml <<'ENDOFFILE'
---
# FALLO 1: imagen inexistente → ImagePullBackOff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fallo-imagen
  namespace: kcna-fallos
spec:
  replicas: 1
  selector:
    matchLabels: { app: fallo-imagen }
  template:
    metadata:
      labels: { app: fallo-imagen }
    spec:
      containers:
      - name: webapp
        image: nginx:no-existe-99
        ports:
        - containerPort: 80
---
# FALLO 2: comando que sale con error → CrashLoopBackOff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fallo-crash
  namespace: kcna-fallos
spec:
  replicas: 1
  selector:
    matchLabels: { app: fallo-crash }
  template:
    metadata:
      labels: { app: fallo-crash }
    spec:
      containers:
      - name: app
        image: busybox:1.36.1
        command: ["/bin/sh", "-c", "echo 'Falta configuración obligatoria'; exit 1"]
---
# FALLO 3: Deployment cuyo selector NO coincide con las labels del template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fallo-selector
  namespace: kcna-fallos
spec:
  replicas: 2
  selector:
    matchLabels: { app: fallo-selector }
  template:
    metadata:
      labels: { app: etiqueta-distinta }   # FALLO: no coincide con el selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
---
# FALLO 4: Service con selector que no matchea ningún Pod → sin endpoints
apiVersion: v1
kind: Service
metadata:
  name: fallo-svc
  namespace: kcna-final
spec:
  selector:
    app: kcna-webapp-INEXISTENTE   # FALLO: los Pods son app=kcna-webapp
  ports:
  - port: 8080
    targetPort: 8080
---
# FALLO 5: Deployment que referencia una key de Secret que no existe
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fallo-secret
  namespace: kcna-final
spec:
  replicas: 1
  selector:
    matchLabels: { app: fallo-secret }
  template:
    metadata:
      labels: { app: fallo-secret }
    spec:
      containers:
      - name: webapp
        image: busybox:1.36.1
        command: ["/bin/sh", "-c", "sleep 3600"]
        env:
        - name: TOKEN
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: CLAVE_QUE_NO_EXISTE   # FALLO: la key no existe en app-secret
ENDOFFILE

kubectl apply -f ~/kcna-labs/lab05/fallos.yaml
sleep 25
```

**Verificación:** confirma que los fallos se manifiestan:

```bash
kubectl get pods -n kcna-fallos
kubectl get pods -n kcna-final -l app=fallo-secret
kubectl get endpoints fallo-svc -n kcna-final
```

---

### Paso 8 — Escenario 1: ImagePullBackOff

**Instrucciones:**

1. **Observar** y **describir**:

```bash
kubectl get pods -n kcna-fallos -l app=fallo-imagen
POD=$(kubectl get pods -n kcna-fallos -l app=fallo-imagen -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n kcna-fallos | grep -A6 "Events"
```

**Salida esperada:** estado `ImagePullBackOff`; eventos con `Failed to pull image "nginx:no-existe-99"`.

2. **Corregir y verificar** (imagen válida):

```bash
kubectl set image deployment/fallo-imagen webapp=nginx:1.25.4 -n kcna-fallos
kubectl rollout status deployment/fallo-imagen -n kcna-fallos --timeout=90s
kubectl get pods -n kcna-fallos -l app=fallo-imagen
```

> **Causa raíz:** tag de imagen inexistente. **Corrección:** usar un tag válido. Síntoma diferenciador: `ImagePullBackOff` con `RESTARTS=0`.

---

### Paso 9 — Escenario 2: CrashLoopBackOff

**Instrucciones:**

1. **Observar** e **inspeccionar logs**:

```bash
kubectl get pods -n kcna-fallos -l app=fallo-crash
POD=$(kubectl get pods -n kcna-fallos -l app=fallo-crash -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n kcna-fallos --previous 2>/dev/null || kubectl logs $POD -n kcna-fallos
```

**Salida esperada:** `CrashLoopBackOff` con reinicios crecientes; logs muestran `Falta configuración obligatoria` y exit code 1.

2. **Corregir y verificar** (comando que no falla):

```bash
kubectl patch deployment fallo-crash -n kcna-fallos --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/command","value":["/bin/sh","-c","echo OK; sleep 3600"]}
]'
kubectl rollout status deployment/fallo-crash -n kcna-fallos --timeout=90s
kubectl get pods -n kcna-fallos -l app=fallo-crash
```

> **Causa raíz:** el proceso termina con `exit 1`. **Corrección:** proporcionar la configuración/comando válido. Síntoma diferenciador: `CrashLoopBackOff` con logs de error explícito.

---

### Paso 10 — Escenario 3: Selector del Deployment no coincide con el template

**Instrucciones:**

1. **Observar** y **describir** (el Deployment no llega a crear Pods correctamente):

```bash
kubectl get deployment fallo-selector -n kcna-fallos
kubectl apply -f ~/kcna-labs/lab05/fallos.yaml 2>&1 | grep -i "fallo-selector" || true
kubectl describe deployment fallo-selector -n kcna-fallos | grep -A3 -i "selector"
```

**Salida esperada:** el `selector.matchLabels` (`app=fallo-selector`) no coincide con las labels del template (`app=etiqueta-distinta`). En Kubernetes, esto provoca un error de validación (`selector does not match template labels`) al aplicar, o un Deployment que no gestiona sus Pods.

2. **Corregir y verificar** (alinear labels):

```bash
kubectl delete deployment fallo-selector -n kcna-fallos --ignore-not-found

cat > ~/kcna-labs/lab05/fallo-selector-fixed.yaml <<'ENDOFFILE'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fallo-selector
  namespace: kcna-fallos
spec:
  replicas: 2
  selector:
    matchLabels: { app: fallo-selector }
  template:
    metadata:
      labels: { app: fallo-selector }   # ahora coincide con el selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
ENDOFFILE

kubectl apply -f ~/kcna-labs/lab05/fallo-selector-fixed.yaml
kubectl rollout status deployment/fallo-selector -n kcna-fallos --timeout=90s
```

> **Causa raíz:** `selector.matchLabels` ≠ `template.metadata.labels`. **Corrección:** que coincidan. Concepto de examen: el selector de un Deployment **debe** cubrir las labels del template.

---

### Paso 11 — Escenario 4: Service sin endpoints (selector incorrecto)

**Instrucciones:**

1. **Observar** y **diagnosticar**:

```bash
kubectl get endpoints fallo-svc -n kcna-final
kubectl get svc fallo-svc -n kcna-final -o jsonpath='{.spec.selector}'; echo
kubectl get pods -n kcna-final -l app=kcna-webapp --show-labels | head -2
```

**Salida esperada:** `ENDPOINTS <none>`; el selector es `app=kcna-webapp-INEXISTENTE`, que no coincide con la label real `app=kcna-webapp`.

2. **Corregir y verificar**:

```bash
kubectl patch svc fallo-svc -n kcna-final --type='json' \
  -p='[{"op":"replace","path":"/spec/selector/app","value":"kcna-webapp"}]'
sleep 3
kubectl get endpoints fallo-svc -n kcna-final
```

> **Causa raíz:** selector del Service no matchea las labels de los Pods. **Corrección:** alinear el selector. Síntoma diferenciador: Pods sanos pero `ENDPOINTS <none>`.

---

### Paso 12 — Escenario 5: Referencia a una key de Secret inexistente

**Instrucciones:**

1. **Observar** y **describir** (el Pod no arranca por `CreateContainerConfigError`):

```bash
kubectl get pods -n kcna-final -l app=fallo-secret
POD=$(kubectl get pods -n kcna-final -l app=fallo-secret -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n kcna-final | grep -A5 "Events"
```

**Salida esperada:** estado `CreateContainerConfigError`; evento `couldn't find key CLAVE_QUE_NO_EXISTE in Secret kcna-final/app-secret`.

2. **Localizar** las keys reales del Secret y **corregir**:

```bash
kubectl get secret app-secret -n kcna-final -o jsonpath='{.data}' | python3 -c "import sys,json; print('Keys:', list(json.load(sys.stdin).keys()))"

# Corregir la referencia a una key existente (DB_PASSWORD)
kubectl patch deployment fallo-secret -n kcna-final --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/env/0/valueFrom/secretKeyRef/key","value":"DB_PASSWORD"}
]'
kubectl rollout status deployment/fallo-secret -n kcna-final --timeout=90s
kubectl get pods -n kcna-final -l app=fallo-secret
```

> **Causa raíz:** `secretKeyRef.key` apunta a una key que no existe. **Corrección:** referenciar una key existente. Síntoma diferenciador: `CreateContainerConfigError` (no es ImagePull ni CrashLoop).

---

## Parte 3: Autoevaluación — 10 Preguntas Tipo KCNA (20 minutos)

Responde cada pregunta del dominio **Kubernetes Fundamentals** y luego compara con la clave.

1. ¿Cuál es la unidad mínima de despliegue en Kubernetes?
   - a) Contenedor  b) Pod  c) Deployment  d) Node

2. ¿Qué componente del control plane asigna Pods a nodos?
   - a) kubelet  b) etcd  c) kube-scheduler  d) kube-proxy

3. ¿Qué objeto gestiona un conjunto de réplicas idénticas de Pods y permite rolling updates?
   - a) ReplicaSet  b) Deployment  c) Pod  d) DaemonSet

4. Un Service de tipo ClusterIP es accesible...
   - a) desde Internet  b) solo dentro del clúster  c) en un puerto de cada nodo  d) vía DNS externo

5. ¿Dónde se almacena el estado del clúster de Kubernetes?
   - a) kubelet  b) etcd  c) API Server  d) ConfigMap

6. ¿Qué campo conecta un Service con sus Pods?
   - a) annotations  b) selector/labels  c) namespace  d) nodeName

7. Un Pod queda en `Pending`. La causa MÁS probable relacionada con scheduling es...
   - a) imagen inexistente  b) recursos insuficientes / no hay nodo viable  c) readinessProbe falla  d) Service sin endpoints

8. ¿Qué recurso guarda configuración NO sensible en pares clave-valor?
   - a) Secret  b) ConfigMap  c) PVC  d) Role

9. En un manifiesto de Deployment, el `selector.matchLabels` debe...
   - a) ser distinto de las labels del template  b) coincidir con las labels del template del Pod  c) estar vacío  d) igualar el nombre del namespace

10. ¿Qué comando aplica un manifiesto declarativo y reconcilia el estado deseado?
    - a) `kubectl create`  b) `kubectl run`  c) `kubectl apply -f`  d) `kubectl exec`

**Clave de respuestas y justificación:**

```bash
cat > ~/kcna-labs/lab05/respuestas.md <<'ENDOFFILE'
# Clave — Autoevaluación KCNA (Lab 05, dominio Kubernetes Fundamentals)

1. b) Pod — un Pod agrupa uno o más contenedores; es la unidad mínima que Kubernetes programa.
2. c) kube-scheduler — decide en qué nodo corre cada Pod (filtering + scoring).
3. b) Deployment — gestiona ReplicaSets y habilita rolling updates/rollbacks.
4. b) Solo dentro del clúster — ClusterIP es una IP virtual interna.
5. b) etcd — almacén clave-valor que guarda todo el estado del clúster.
6. b) selector/labels — el Service selecciona Pods por sus labels.
7. b) Recursos insuficientes / no hay nodo viable — típico de Pending (fase de filtering del scheduler).
8. b) ConfigMap — configuración no sensible; los Secrets son para datos sensibles.
9. b) Debe coincidir con las labels del template del Pod (si no, error de validación).
10. c) kubectl apply -f — modelo declarativo que reconcilia el estado deseado.
ENDOFFILE

cat ~/kcna-labs/lab05/respuestas.md
```

Calcula tu puntuación (X/10). Una puntuación ≥ 8/10 indica buen dominio del dominio Kubernetes Fundamentals.

---

## Validación y Pruebas Finales

Ejecuta el script para confirmar que las tres partes se completaron:

```bash
cat > ~/kcna-labs/lab05/validate.sh <<'SCRIPT'
#!/bin/bash
echo "=== Validación Final - Lab 05 (Integrador) ==="
PASS=0; FAIL=0
check() { if eval "$2"; then echo "✅ $1"; ((PASS++)); else echo "❌ $1"; ((FAIL++)); fi; }

# Parte 1: arquitectura
PROD=$(kubectl get deploy kcna-webapp -n kcna-final -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
STG=$(kubectl get deploy kcna-webapp-staging -n kcna-staging -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "P1: producción 3/3 (actual: $PROD)" "[ \"$PROD\" = \"3\" ]"
check "P1: staging 2/2 (actual: $STG)" "[ \"$STG\" = \"2\" ]"
check "P1: Service de producción tiene endpoints" "[ -n \"$(kubectl get endpoints kcna-webapp-svc -n kcna-final -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)\" ]"

# Parte 2: fallos resueltos
check "P2: fallo-imagen Running" "[ \"$(kubectl get pods -n kcna-fallos -l app=fallo-imagen -o jsonpath='{.items[0].status.phase}' 2>/dev/null)\" = Running ]"
check "P2: fallo-crash Running" "[ \"$(kubectl get pods -n kcna-fallos -l app=fallo-crash -o jsonpath='{.items[0].status.phase}' 2>/dev/null)\" = Running ]"
check "P2: fallo-selector 2/2" "[ \"$(kubectl get deploy fallo-selector -n kcna-fallos -o jsonpath='{.status.readyReplicas}' 2>/dev/null)\" = \"2\" ]"
check "P2: fallo-svc con endpoints" "[ -n \"$(kubectl get endpoints fallo-svc -n kcna-final -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)\" ]"
check "P2: fallo-secret Running" "[ \"$(kubectl get pods -n kcna-final -l app=fallo-secret -o jsonpath='{.items[0].status.phase}' 2>/dev/null)\" = Running ]"

# Parte 3: autoevaluación
check "P3: clave de respuestas generada" "test -f ~/kcna-labs/lab05/respuestas.md"

echo ""
echo "=== Resultado: $PASS/9 pruebas exitosas, $FAIL fallidas ==="
if [ $FAIL -eq 0 ]; then echo "🎉 ¡Laboratorio integrador completado!"; fi
SCRIPT

chmod +x ~/kcna-labs/lab05/validate.sh
bash ~/kcna-labs/lab05/validate.sh
```

**Salida esperada:** las 9 pruebas deben mostrar `✅`.

---

## Solución de Problemas

### Problema 1: Los Pods de producción/staging quedan en `ImagePullBackOff`

**Síntomas:** los Pods de `kcna-webapp` no arrancan; `describe` muestra fallo al descargar la imagen.

**Causa:** no se reemplazó `youruser` por tu usuario real de Docker Hub, o la imagen no está cargada en Minikube.

**Solución:**

```bash
# Verificar la imagen configurada
kubectl get deploy kcna-webapp -n kcna-final -o jsonpath='{.spec.template.spec.containers[0].image}'; echo

# Si es local, cargarla en Minikube
minikube image load ${DOCKERHUB_USER:-tu-usuario}/kcna-webapp:1.0.0

# O corregir el manifiesto y reaplicar
sed -i "s/youruser/${DOCKERHUB_USER}/g" ~/kcna-labs/lab05/deployment-production.yaml
kubectl apply -f ~/kcna-labs/lab05/deployment-production.yaml
```

### Problema 2: Los Pods de staging quedan en `Pending`

**Síntomas:** los Pods de `kcna-webapp-staging` no se programan.

**Causa:** el `nodeSelector environment=staging` no encuentra ningún nodo con esa label (por ejemplo, en un clúster de un solo nodo `minikube-m02` no existe).

**Solución:**

```bash
# Ver el evento de scheduling
kubectl describe pod -n kcna-staging -l app=kcna-webapp | grep -A5 "Events"

# En clúster de un solo nodo, etiquetar el nodo principal en su lugar
kubectl label node minikube environment=staging --overwrite
# o eliminar el nodeSelector del Deployment si no hay nodos worker
```

---

## Limpieza

```bash
# Eliminar los namespaces del laboratorio (incluye producción, staging y fallos)
kubectl delete namespace kcna-final kcna-staging kcna-fallos

# Quitar la label del nodo usada por staging
kubectl label node minikube-m02 environment- 2>/dev/null
kubectl label node minikube environment- 2>/dev/null

# Verificar limpieza
kubectl get ns | grep -E "kcna-final|kcna-staging|kcna-fallos" || echo "Namespaces eliminados correctamente"
```

> **Nota:** Si vas a continuar con el Capítulo 6 (Networking), puedes conservar el clúster Minikube.

---

## Resumen

Has completado el **laboratorio integrador de fundamentos** en tres fases:

| Parte | Actividad | Resultado |
|-------|-----------|-----------|
| 1 | Reconstrucción de arquitectura | Multi-namespace (producción + staging), ConfigMap/Secret, Deployments con probes, Services (NodePort/ClusterIP), nodeSelector |
| 2 | Diagnóstico de 5 fallos | ImagePullBackOff, CrashLoopBackOff, selector≠template, Service sin endpoints, key de Secret inexistente |
| 3 | Autoevaluación KCNA | 10 preguntas del dominio Kubernetes Fundamentals con clave |

### Conceptos Clave Reforzados

- Los **namespaces** aíslan lógicamente entornos; los **Services** conectan con Pods por **labels/selectors**.
- Cada estado de fallo tiene un **síntoma diferenciador**: `ImagePullBackOff` (imagen), `CrashLoopBackOff` (proceso que sale con error), `CreateContainerConfigError` (ConfigMap/Secret mal referenciado), `Pending` (scheduling), `ENDPOINTS <none>` (selector de Service).
- El **selector del Deployment debe coincidir** con las labels del template del Pod.
- El **modelo declarativo** (`kubectl apply`) reconcilia el estado deseado y es la base de CI/CD y GitOps.

### Recursos Adicionales

- [Kubernetes Fundamentals — Conceptos](https://kubernetes.io/docs/concepts/)
- [Debug Pods — Kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [KCNA Curriculum (CNCF)](https://github.com/cncf/curriculum)
- [CNCF Cloud Native Glossary (ES)](https://glossary.cncf.io/es/)

### Próximo Capítulo

En el **Capítulo 6 (Práctica 6)** profundizarás en networking: tipos de Service, Ingress, DNS interno con CoreDNS y NetworkPolicies.
