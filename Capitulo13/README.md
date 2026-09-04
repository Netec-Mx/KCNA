# 15 Práctica 13. Flujo cloud native completo para KCNA

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 120 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear / Evaluar |
| **Módulo** | 13 – Arquitectura, ecosistema CNCF y laboratorio final integrador |

## Descripción General

Este es el **laboratorio final integrador** del curso. En una sola sesión aplicarás, de principio a fin, casi todos los dominios de KCNA sobre la aplicación `kcna-webapp`: construirás la arquitectura cloud native completa (Namespace, ConfigMap, Secret, Deployment con probes, Service), aplicarás **seguridad** (RBAC de mínimo privilegio y SecurityContext), demostrarás **entrega de aplicaciones** (rolling update + rollback), habilitarás **observabilidad** (métricas y logs), practicarás **debugging** de un fallo intencional, mapearás los componentes desplegados contra el **CNCF Landscape** y los **principios cloud native**, y cerrarás con un **cuestionario de 10 preguntas tipo KCNA** con autoevaluación. Trabajarás en cinco fases documentadas en `~/kcna-labs/lab13/`.

## Presupuesto de Tiempo (120 minutos)

Este laboratorio está dimensionado para **120 minutos de trabajo efectivo**, con el siguiente reparto por fase:

| Fase | Contenido | Tiempo |
|------|-----------|--------|
| Fase 1 | Arquitectura y configuración (Pasos 1–2) | 25 min |
| Fase 2 | Seguridad — RBAC de mínimo privilegio (Paso 3) | 20 min |
| Fase 3 | Entrega — rolling update y rollback (Paso 4) | 20 min |
| Fase 4 | Observabilidad y debugging (Pasos 5–6) | 25 min |
| Fase 5 | Arquitectura CNCF y autoevaluación (Pasos 7–8) | 30 min |
| **Total** | | **120 min** |

> **⚠️ IMPORTANTE — realiza estos prerrequisitos ANTES de arrancar el cronómetro de 120 min.** Las siguientes tareas son lentas (descargas de red) y **no** están contadas en el presupuesto de tiempo. Complétalas en la fase de "Preparación del entorno":
> 1. `minikube start` (arranque del clúster: 1–3 min si es nuevo).
> 2. `minikube addons enable metrics-server` y esperar a que recolecte datos (~1–2 min).
> 3. **Pre-cargar las imágenes** que usa la Fase 3 para no perder tiempo durante el ejercicio:
>    ```bash
>    docker pull docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
>    docker tag docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0 docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
>    minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
>    minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
>    ```
> Con estos prerrequisitos listos, las 5 fases se completan cómodamente en 120 min. Si arrancas "en frío" (sin clúster ni imágenes), suma 10–15 min extra de preparación.

## Objetivos de Aprendizaje

- [ ] Integrar arquitectura, configuración, seguridad, entrega y observabilidad en un único flujo cloud native reproducible
- [ ] Aplicar RBAC de mínimo privilegio y un SecurityContext no-root sobre una aplicación desplegada
- [ ] Demostrar una actualización con rolling update y su rollback verificando la ausencia de downtime
- [ ] Habilitar observabilidad (métricas con metrics-server y logs) y correlacionarla con las señales de salud del clúster
- [ ] Diagnosticar y resolver un fallo intencional aplicando la metodología de troubleshooting/debugging
- [ ] Relacionar los objetos y herramientas del laboratorio con el CNCF Landscape y los principios cloud native, y autoevaluar conceptos con preguntas tipo KCNA

## Prerrequisitos

### Conocimiento previo

| Requisito | Descripción |
|-----------|-------------|
| Labs 01–12 completados | Dominio de contenedores, objetos K8s, kubectl, scheduling, networking, RBAC, storage, troubleshooting, delivery, debugging y observabilidad |
| Imagen publicada | `[dockerhub-user]/kcna-webapp:1.0.0` disponible en Docker Hub |
| Manifiestos YAML | Lectura y escritura fluida de manifiestos declarativos |
| Conceptos CNCF | Principios cloud native, CNCF Landscape y proyectos clave (Capítulo 13) |

### Acceso requerido

- Clúster Minikube en ejecución con driver Docker
- `kubectl` configurado con acceso de administrador al clúster
- Conexión a Internet para descargar imágenes

## Entorno del Laboratorio

### Software necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.33.1 | Clúster Kubernetes local |
| Kubernetes | 1.30.0 | Orquestador (vía Minikube) |
| kubectl | 1.30.2 | Cliente CLI de Kubernetes |
| metrics-server | addon Minikube | Métricas de recursos (observabilidad) |
| Docker Engine | 26.1.4 | Runtime de contenedores |
| curl | 8.8.0 | Verificación de endpoints HTTP |

> **Nota de versiones:** Se usa Kubernetes 1.30.0 / kubectl 1.30.2. Si tu clúster viene de labs 6–9 (1.29.2), el laboratorio es compatible; ajusta la versión en `minikube start`.

### Preparación del entorno

```bash
mkdir -p ~/kcna-labs/lab13
cd ~/kcna-labs/lab13

export DOCKERHUB_USER=<tu-usuario-dockerhub>
echo "Usuario Docker Hub: $DOCKERHUB_USER"

# Iniciar clúster (o reutilizar el existente)
minikube start --kubernetes-version=v1.30.0 --driver=docker --cpus=2 --memory=4096

# Habilitar métricas para la fase de observabilidad
minikube addons enable metrics-server

# PRE-CARGA de imágenes (fuera del cronómetro de 120 min): evita esperas en la Fase 3
docker pull docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
docker tag  docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0 docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0

kubectl get nodes
```

**Salida esperada:**

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   v1.30.0
```

---

## Fase 1: Arquitectura y configuración (25 min)

### Paso 1 — Namespace, ConfigMap y Secret

**Objetivo:** Establecer la base de la arquitectura: un namespace aislado con configuración externalizada (ConfigMap) y credenciales protegidas (Secret).

**Instrucciones:**

1. Crea el namespace y los recursos de configuración:

```bash
kubectl create namespace kcna-final

cat > ~/kcna-labs/lab13/01-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: kcna-final
data:
  APP_ENVIRONMENT: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: kcna-final
type: Opaque
stringData:
  APP_SECRET_KEY: "kcna-final-lab-secret"
EOF

kubectl apply -f ~/kcna-labs/lab13/01-config.yaml
```

**Salida esperada:**

```
namespace/kcna-final created
configmap/app-config created
secret/app-secret created
```

**Verificación:**

```bash
kubectl get configmap,secret -n kcna-final
```

Debe listar `app-config` y `app-secret`.

---

### Paso 2 — Deployment con probes, SecurityContext y Service

**Objetivo:** Desplegar la aplicación con 3 réplicas, probes de salud, ejecución no-root (SecurityContext) e inyección de configuración, y exponerla con un Service NodePort.

**Instrucciones:**

1. Crea el manifiesto del Deployment seguro y su Service:

```bash
cat > ~/kcna-labs/lab13/02-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-final
  labels:
    app: kcna-webapp
  annotations:
    kubernetes.io/change-cause: "Despliegue inicial v1.0.0"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: kcna-webapp
  template:
    metadata:
      labels:
        app: kcna-webapp
        version: "1.0.0"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: APP_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: APP_SECRET_KEY
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
          capabilities:
            drop: ["ALL"]
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "300m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
---
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
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
EOF

kubectl apply -f ~/kcna-labs/lab13/02-deployment.yaml
kubectl rollout status deployment/kcna-webapp -n kcna-final --timeout=120s
```

> **Nota:** La imagen `kcna-webapp` corre como usuario `appuser` (UID definido en su Dockerfile del Lab 01). Si tu imagen no fija un UID no-root compatible con `runAsUser: 1000`, ajústalo o elimina `runAsUser` dejando solo `runAsNonRoot: true`.

2. Verifica el despliegue y la conectividad:

```bash
kubectl get pods -n kcna-final -l app=kcna-webapp -o wide
SVC_URL=$(minikube service kcna-webapp-svc -n kcna-final --url)
curl -s $SVC_URL/ | python3 -m json.tool
```

**Salida esperada:** 3 Pods `Running`/`Ready` y un JSON con `version: "1.0.0"` y `environment: production`.

**Verificación:**

```bash
# Confirmar que la app corre como no-root
POD=$(kubectl get pods -n kcna-final -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n kcna-final -- id
```

Debe mostrar un UID distinto de 0 (no root).

---

## Fase 2: Seguridad — RBAC de mínimo privilegio (20 min)

### Paso 3 — ServiceAccount, Role y RoleBinding de solo lectura

**Objetivo:** Crear una identidad de solo lectura para la aplicación y verificar el principio de mínimo privilegio.

**Instrucciones:**

1. Crea los recursos RBAC:

```bash
cat > ~/kcna-labs/lab13/03-rbac.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-reader
  namespace: kcna-final
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: reader-role
  namespace: kcna-final
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: reader-binding
  namespace: kcna-final
subjects:
- kind: ServiceAccount
  name: webapp-reader
  namespace: kcna-final
roleRef:
  kind: Role
  name: reader-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f ~/kcna-labs/lab13/03-rbac.yaml
```

2. Verifica los permisos concedidos y denegados:

```bash
SA=system:serviceaccount:kcna-final:webapp-reader
echo -n "list pods (kcna-final): ";     kubectl auth can-i list pods   -n kcna-final --as=$SA
echo -n "get configmaps (kcna-final): "; kubectl auth can-i get configmaps -n kcna-final --as=$SA
echo -n "delete pods (kcna-final): ";    kubectl auth can-i delete pods -n kcna-final --as=$SA
echo -n "list secrets (kcna-final): ";   kubectl auth can-i list secrets -n kcna-final --as=$SA
echo -n "list pods (default): ";         kubectl auth can-i list pods   -n default    --as=$SA
```

**Salida esperada:**

```
list pods (kcna-final): yes
get configmaps (kcna-final): yes
delete pods (kcna-final): no
list secrets (kcna-final): no
list pods (default): no
```

**Verificación:** Los permisos coinciden exactamente con lo definido (lectura de pods/configmaps solo en `kcna-final`). Esto demuestra el principio de **mínimo privilegio**.

---

## Fase 3: Entrega de aplicaciones — rolling update y rollback (20 min)

### Paso 4 — Actualización a v2.0.0 y rollback

**Objetivo:** Ejecutar una actualización sin downtime y revertirla, cerrando el dominio de Application Delivery.

**Instrucciones:**

1. Actualiza a la imagen v2.0.0 (ya pre-cargada en la preparación del entorno, por lo que el rollout es inmediato):

```bash
# La imagen 2.0.0 ya fue cargada en Minikube durante la preparación.
# Si por algún motivo no lo hiciste, ejecútalo ahora (tarda 1-3 min):
#   docker tag docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0 docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
#   minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0

kubectl set image deployment/kcna-webapp \
  kcna-webapp=docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0 -n kcna-final
kubectl annotate deployment/kcna-webapp -n kcna-final \
  kubernetes.io/change-cause="Actualización a v2.0.0" --overwrite
kubectl rollout status deployment/kcna-webapp -n kcna-final --timeout=120s
```

2. Verifica el historial y luego revierte:

```bash
kubectl rollout history deployment/kcna-webapp -n kcna-final
kubectl rollout undo deployment/kcna-webapp -n kcna-final
kubectl rollout status deployment/kcna-webapp -n kcna-final --timeout=120s
```

**Salida esperada:** El historial muestra revisiones 1 (inicial), 2 (v2.0.0) y 3 (rollback). Tras el `undo`, los Pods vuelven a la imagen previa.

**Verificación:**

```bash
kubectl get pods -n kcna-final -l app=kcna-webapp \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}' | sort -u
# El servicio nunca dejó de responder gracias a maxUnavailable: 0
curl -s -o /dev/null -w "HTTP durante el proceso: %{http_code}\n" $(minikube service kcna-webapp-svc -n kcna-final --url)/health
```

Debe devolver `HTTP: 200`.

---

## Fase 4: Observabilidad y debugging (25 min)

### Paso 5 — Métricas y logs

**Objetivo:** Consultar métricas y logs de la aplicación desplegada, generando carga para observar la variación.

**Instrucciones:**

1. Consulta métricas (espera ~30 s tras el rollout):

```bash
sleep 30
kubectl top pods -n kcna-final
```

2. Genera carga y observa logs y métricas:

```bash
kubectl run load-gen --rm -it --restart=Never \
  --image=busybox:1.36.1 -n kcna-final -- /bin/sh -c '
    end=$(( $(date +%s) + 30 ));
    while [ $(date +%s) -lt $end ]; do
      wget -q -O /dev/null http://kcna-webapp-svc.kcna-final.svc.cluster.local:8080/ 2>/dev/null;
    done; echo "Carga finalizada."'

POD=$(kubectl get pods -n kcna-final -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n kcna-final --since=2m | grep -c '"GET / HTTP/1.1" 200'
```

**Salida esperada:** `kubectl top pods` lista los Pods con su consumo; el conteo de logs de peticiones GET 200 es mayor que 0.

---

### Paso 6 — Debugging de un fallo intencional

**Objetivo:** Introducir un fallo (Service con selector erróneo), diagnosticarlo con la metodología del curso y corregirlo.

**Instrucciones:**

1. Rompe el selector del Service de forma intencional:

```bash
kubectl patch svc kcna-webapp-svc -n kcna-final --type='json' \
  -p='[{"op":"replace","path":"/spec/selector/app","value":"kcna-webapp-typo"}]'
```

2. **Observar** y **diagnosticar**:

```bash
# El Service se queda sin endpoints
kubectl get endpoints kcna-webapp-svc -n kcna-final
# Confirmar el selector erróneo
kubectl get svc kcna-webapp-svc -n kcna-final -o jsonpath='{.spec.selector}'; echo
# Comparar con los labels reales de los Pods
kubectl get pods -n kcna-final -l app=kcna-webapp --show-labels
```

**Salida esperada:** `ENDPOINTS <none>` y el selector `{"app":"kcna-webapp-typo"}` que no coincide con la label real `app=kcna-webapp`.

3. **Actuar** y **verificar**:

```bash
kubectl patch svc kcna-webapp-svc -n kcna-final --type='json' \
  -p='[{"op":"replace","path":"/spec/selector/app","value":"kcna-webapp"}]'
sleep 3
kubectl get endpoints kcna-webapp-svc -n kcna-final
```

**Verificación:** Los endpoints vuelven a poblarse con las IPs de los 3 Pods.

> **Análisis:** Un Service sin endpoints por selector erróneo es un fallo clásico: los Pods están sanos (`Running`, `Ready`) pero el Service no los encuentra. El síntoma diferenciador es `ENDPOINTS <none>` con Pods sanos.

---

## Fase 5: Arquitectura CNCF y autoevaluación (30 min)

### Paso 7 — Mapear el laboratorio contra el CNCF Landscape y los principios cloud native

**Objetivo:** Consolidar la visión de arquitectura relacionando lo desplegado con las categorías del CNCF Landscape y los principios cloud native.

**Instrucciones:**

1. Genera el documento de mapeo:

```bash
cat > ~/kcna-labs/lab13/mapeo-cncf.md << 'EOF'
# Mapeo del laboratorio contra CNCF y principios cloud native

## Objetos usados → categoría CNCF Landscape → proyecto clave
| En este lab               | Categoría CNCF                 | Proyecto(s) CNCF de referencia        |
|---------------------------|--------------------------------|---------------------------------------|
| Contenedor kcna-webapp    | Container Runtime              | containerd, CRI-O (runtime); OCI (std)|
| Kubernetes (Minikube)     | Orchestration & Management     | Kubernetes                            |
| Service / DNS interno     | Service Discovery / Networking | CoreDNS; CNI (Calico/Cilium)          |
| Deployment/rollout        | App Definition & Delivery      | Helm, Kustomize, Argo CD, Flux        |
| RBAC / SecurityContext    | Security & Compliance          | (modelo K8s); Falco, OPA/Gatekeeper   |
| Métricas (metrics-server) | Observability                  | Prometheus, Grafana                   |
| Logs                      | Observability                  | Fluentd / Fluent Bit                  |
| Trazas (conceptual)       | Observability                  | OpenTelemetry, Jaeger                 |

## Principios cloud native demostrados
- Microservicios: app desacoplada, empaquetada en contenedor.
- Infraestructura inmutable: se reemplazan Pods, no se parchean en caliente.
- Declarative APIs: todo el estado se define en YAML (`kubectl apply`).
- Automatización y self-healing: el Deployment recrea Pods; las probes reinician contenedores no sanos.
- Escalabilidad: réplicas + HPA (basado en métricas del metrics-server).
- Resiliencia: rolling update sin downtime, rollback ante fallos.
- Portabilidad: imagen OCI ejecutable en cualquier runtime conforme.

## Roles de la industria (Cap. 13)
developer · operator · platform engineer · SRE · security engineer
EOF

cat ~/kcna-labs/lab13/mapeo-cncf.md
```

**Verificación:**

```bash
test -f ~/kcna-labs/lab13/mapeo-cncf.md && echo "✅ Documento de mapeo CNCF generado"
```

---

### Paso 8 — Autoevaluación: 10 preguntas tipo KCNA

**Objetivo:** Repasar conceptos de los cuatro dominios de KCNA con preguntas de opción múltiple y su justificación.

Lee cada pregunta, elige tu respuesta y luego compárala con la clave al final.

1. ¿Qué componente del control plane es el "almacén de la verdad" del estado del clúster?
   - a) kube-scheduler  b) etcd  c) kubelet  d) kube-proxy

2. ¿Qué tipo de Service asigna un puerto en cada nodo para acceso externo?
   - a) ClusterIP  b) NodePort  c) ExternalName  d) Headless

3. Una readinessProbe que falla provoca que el Pod...
   - a) se reinicie  b) se elimine  c) sea retirado de los Endpoints del Service  d) escale a más réplicas

4. ¿Qué recurso solicita almacenamiento persistente en nombre de un Pod?
   - a) PersistentVolume  b) PersistentVolumeClaim  c) StorageClass  d) Volume

5. En RBAC, ¿qué objeto vincula un Role a un sujeto dentro de un namespace?
   - a) ClusterRoleBinding  b) RoleBinding  c) ServiceAccount  d) Role

6. ¿Qué estrategia de despliegue reemplaza los Pods de forma progresiva sin downtime?
   - a) Recreate  b) RollingUpdate  c) Canary manual  d) Blue/Green

7. ¿Cuál de estos es un proyecto CNCF para observabilidad de métricas?
   - a) Fluentd  b) Prometheus  c) Envoy  d) Helm

8. ¿Qué estándar define el formato de imágenes y runtimes de contenedor?
   - a) CRI  b) CNI  c) OCI  d) CSI

9. El principio cloud native de "self-healing" en Kubernetes se manifiesta cuando...
   - a) el usuario reinicia Pods manualmente  b) un Deployment recrea automáticamente un Pod caído  c) se aumenta la RAM del nodo  d) se edita etcd a mano

10. ¿Qué herramienta implementa el patrón GitOps sincronizando el estado del clúster con un repositorio Git?
    - a) kubectl  b) Docker  c) Argo CD  d) containerd

**Instrucciones:** anota tus respuestas y verifica con la clave.

```bash
cat > ~/kcna-labs/lab13/respuestas.md << 'EOF'
# Clave de respuestas — Autoevaluación KCNA (Lab 13)

1. b) etcd — almacén clave-valor del estado del clúster.
2. b) NodePort — expone un puerto en todos los nodos.
3. c) Retirado de los Endpoints — readiness controla tráfico, no reinicio (eso es liveness).
4. b) PersistentVolumeClaim — el PVC es la "solicitud"; el PV es el recurso físico.
5. b) RoleBinding — vincula Role↔sujeto dentro de un namespace (ClusterRoleBinding es cluster-wide).
6. b) RollingUpdate — estrategia por defecto, sin downtime.
7. b) Prometheus — métricas (Fluentd=logs, Envoy=proxy/mesh, Helm=packaging).
8. c) OCI — Open Container Initiative (CRI/CNI/CSI son interfaces de K8s).
9. b) Un Deployment recrea automáticamente un Pod caído.
10. c) Argo CD — herramienta GitOps (también Flux).
EOF

cat ~/kcna-labs/lab13/respuestas.md
```

**Verificación:** calcula tu puntuación (X/10). Una puntuación ≥ 8/10 indica una buena preparación para el examen KCNA en estos temas.

---

## Validación y Pruebas Finales

```bash
cat > ~/kcna-labs/lab13/validate.sh << 'SCRIPT'
#!/bin/bash
echo "=== Validación Final - Lab 13 (Integrador) ==="
PASS=0; FAIL=0
NS=kcna-final
SA=system:serviceaccount:kcna-final:webapp-reader

check() { if eval "$2"; then echo "✅ $1"; ((PASS++)); else echo "❌ $1"; ((FAIL++)); fi; }

# Fase 1: arquitectura
READY=$(kubectl get deployment kcna-webapp -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "F1: Deployment kcna-webapp 3/3 (actual: $READY)" "[ \"$READY\" = \"3\" ]"
check "F1: ConfigMap y Secret existen" "kubectl get configmap app-config -n $NS >/dev/null 2>&1 && kubectl get secret app-secret -n $NS >/dev/null 2>&1"

# Fase 1: seguridad (no-root)
POD=$(kubectl get pods -n $NS -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
UID_VAL=$(kubectl exec $POD -n $NS -- id -u 2>/dev/null)
check "F1: contenedor corre no-root (uid: $UID_VAL)" "[ -n \"$UID_VAL\" ] && [ \"$UID_VAL\" != \"0\" ]"

# Fase 2: RBAC mínimo privilegio
LP=$(kubectl auth can-i list pods -n $NS --as=$SA 2>/dev/null)
DS=$(kubectl auth can-i delete pods -n $NS --as=$SA 2>/dev/null)
LS=$(kubectl auth can-i list secrets -n $NS --as=$SA 2>/dev/null)
check "F2: RBAC lee pods=yes, borra=no, secrets=no (l:$LP d:$DS s:$LS)" "[ \"$LP\" = yes ] && [ \"$DS\" = no ] && [ \"$LS\" = no ]"

# Fase 3: rollout con >=3 revisiones
REVS=$(kubectl rollout history deployment/kcna-webapp -n $NS 2>/dev/null | grep -c '^[0-9]')
check "F3: historial de rollout >=3 revisiones (actual: $REVS)" "[ \"$REVS\" -ge 3 ]"

# Fase 4: métricas y logs
check "F4: kubectl top pods funciona" "kubectl top pods -n $NS >/dev/null 2>&1"
LINES=$(kubectl logs $POD -n $NS --since=30m 2>/dev/null | grep -c 'GET')
check "F4: logs con peticiones GET (actual: $LINES)" "[ \"$LINES\" -ge 1 ]"

# Fase 4: Service con endpoints (tras corregir el selector)
EP=$(kubectl get endpoints kcna-webapp-svc -n $NS -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)
check "F4: Service con endpoints tras debugging (ip: $EP)" "[ -n \"$EP\" ]"

# Fase 5: documentos generados
check "F5: mapeo-cncf.md y respuestas.md generados" "test -f ~/kcna-labs/lab13/mapeo-cncf.md && test -f ~/kcna-labs/lab13/respuestas.md"

echo ""
echo "=== Resultado: $PASS/9 pruebas exitosas, $FAIL fallidas ==="
if [ $FAIL -eq 0 ]; then echo "🎉 ¡Laboratorio final completado! Estás listo para repasar KCNA."; fi
SCRIPT

chmod +x ~/kcna-labs/lab13/validate.sh
bash ~/kcna-labs/lab13/validate.sh
```

**Salida esperada:** las 9 pruebas deben mostrar `✅`.

---

## Solución de Problemas

### Problema 1: El Pod no arranca por `runAsUser: 1000` incompatible con la imagen

**Síntomas:** El Pod queda en `CrashLoopBackOff` o `Error` con mensajes de permisos, o `container has runAsNonRoot and image will run as root`.

**Causa:** El `securityContext` fuerza `runAsUser: 1000`/`runAsNonRoot: true`, pero la imagen no está construida para ese UID o intenta escribir en rutas sin permiso.

**Solución:**

```bash
# Ver el error exacto
kubectl describe pod $POD -n kcna-final | grep -A5 "Events"

# Opción A: quitar runAsUser fijo, dejar solo runAsNonRoot (la imagen del Lab 01 ya usa appuser no-root)
kubectl patch deployment kcna-webapp -n kcna-final --type='json' \
  -p='[{"op":"remove","path":"/spec/template/spec/securityContext/runAsUser"}]'

# Opción B: si la app necesita escribir, asegúrate de readOnlyRootFilesystem: false (ya es el caso)
kubectl rollout status deployment/kcna-webapp -n kcna-final
```

### Problema 2: `kubectl top` no devuelve datos en la Fase 4

**Síntomas:** `error: Metrics API not available`.

**Causa:** El metrics-server no ha recolectado datos todavía o no está habilitado.

**Solución:**

```bash
minikube addons enable metrics-server
kubectl wait --for=condition=Ready pod -l k8s-app=metrics-server -n kube-system --timeout=120s
sleep 60
kubectl top pods -n kcna-final
```

---

## Limpieza

```bash
# Eliminar todos los recursos del laboratorio
kubectl delete namespace kcna-final

# (Opcional) Eliminar la imagen v2.0.0 local
docker rmi docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0 2>/dev/null

# Verificar limpieza
kubectl get ns | grep kcna-final || echo "Namespace eliminado correctamente"
```

Si ya no necesitas el clúster tras finalizar el curso:

```bash
minikube delete
```

---

## Resumen

Has completado el **laboratorio final integrador** cubriendo los cuatro dominios de KCNA en un único flujo:

| Fase | Dominio KCNA | Lo que aplicaste |
|------|--------------|------------------|
| 1 | Kubernetes Fundamentals + Orchestration | Namespace, ConfigMap, Secret, Deployment con probes, Service, SecurityContext |
| 2 | Cloud Native Architecture (seguridad) | RBAC de mínimo privilegio, ejecución no-root |
| 3 | Cloud Native Application Delivery | Rolling update sin downtime + rollback |
| 4 | Delivery + Observability + Troubleshooting | Métricas (metrics-server), logs, debugging de Service |
| 5 | Cloud Native Architecture (CNCF) | Mapeo al CNCF Landscape, principios cloud native, 10 preguntas KCNA |

### Conceptos Clave Reforzados

- Un flujo cloud native completo integra **arquitectura, configuración, seguridad, entrega y observabilidad** de forma **declarativa**.
- El **principio de mínimo privilegio** (RBAC) y la **ejecución no-root** (SecurityContext) son controles de seguridad esenciales.
- **RollingUpdate + rollback** entregan cambios sin downtime y con recuperación segura.
- **Métricas + logs (+ trazas)** son los tres pilares de la **observabilidad**; el metrics-server alimenta el HPA.
- El **CNCF Landscape** organiza el ecosistema; Kubernetes, Prometheus, Envoy, Fluentd, containerd, Helm, Argo/Flux son proyectos clave.

### Recursos Adicionales

- [KCNA — Curriculum oficial (CNCF)](https://github.com/cncf/curriculum)
- [CNCF Landscape](https://landscape.cncf.io/)
- [CNCF Cloud Native Glossary (ES)](https://glossary.cncf.io/es/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Cloud Native Definition (CNCF TOC)](https://github.com/cncf/toc/blob/main/DEFINITION.md)

### Cierre del Curso

Con este laboratorio has recorrido el ciclo completo: contenedores → fundamentos K8s → administración → scheduling → networking → seguridad → storage → troubleshooting → entrega → debugging → observabilidad → arquitectura CNCF. ¡Éxito en tu certificación KCNA!
