# 13 Práctica 10. Despliegue, actualización y rollback

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 75 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Deployment, RollingUpdate, Recreate, ConfigMap, Secret, `kubectl rollout`, `kubectl diff`, Kustomize |

## Descripción General

En este laboratorio aplicarás los conceptos de entrega de aplicaciones cloud native (Application Delivery) de forma práctica. Desplegarás la aplicación `kcna-webapp` mediante manifiestos declarativos, ejecutarás una actualización de versión con estrategia **RollingUpdate** observando el proceso de reemplazo progresivo de Pods, provocarás una actualización fallida y ejecutarás un **rollback** a la revisión estable, contrastarás la estrategia **Recreate** frente a **RollingUpdate**, gestionarás configuración externa con ConfigMaps y Secrets, y finalmente conocerás Kustomize como herramienta declarativa de personalización de manifiestos. El laboratorio recorre el ciclo completo de empaquetar → desplegar → actualizar → operar.

## Objetivos de Aprendizaje

- [ ] Desplegar la aplicación con un Deployment declarativo y registrar la causa de cada cambio (`change-cause`) para el historial de rollout
- [ ] Ejecutar una actualización de imagen con estrategia RollingUpdate observando `maxSurge` y `maxUnavailable` en acción
- [ ] Provocar una actualización fallida (imagen inexistente) y ejecutar un rollback a una revisión anterior con `kubectl rollout undo`
- [ ] Contrastar el comportamiento de las estrategias RollingUpdate y Recreate en cuanto a disponibilidad durante el despliegue
- [ ] Externalizar la configuración con ConfigMaps/Secrets y comprender cómo un cambio de configuración dispara (o no) un nuevo rollout
- [ ] Reconocer el rol de Kustomize para generar variantes de manifiestos sin duplicar YAML

## Prerrequisitos

### Conocimientos previos

- Laboratorios 01–03 completados: imagen `[dockerhub-user]/kcna-webapp:1.0.0` publicada en Docker Hub
- Comprensión de Deployments, ReplicaSets, ConfigMaps y Secrets (Capítulos 2 y 3)
- Familiaridad con `kubectl apply`, `get`, `describe`, `rollout` (Capítulo 3)
- Conceptos de CI/CD y GitOps a nivel introductorio (Capítulo 10, lecciones 10.1–10.10)

### Acceso requerido

- Clúster Minikube en ejecución con driver Docker
- `kubectl` configurado y apuntando al clúster
- Imagen `[dockerhub-user]/kcna-webapp:1.0.0` accesible (o cargable con `minikube image load`)
- Conexión a Internet para descargar imágenes

## Entorno del Laboratorio

### Software necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.33.1 | Clúster Kubernetes local |
| Kubernetes | 1.30.0 | Orquestador (vía Minikube) |
| kubectl | 1.30.2 | Cliente CLI de Kubernetes |
| Docker Engine | 26.1.4 | Runtime de contenedores |
| curl | 8.8.0 | Verificación de endpoints HTTP |

> **Nota de versiones:** Este laboratorio usa Kubernetes 1.30.0 / kubectl 1.30.2 (alineado con los laboratorios 1–5). Si tu entorno viene de los laboratorios 6–9 (Kubernetes 1.29.2), el laboratorio es igualmente compatible; ajusta la versión en `minikube start` según corresponda.

### Preparación del entorno

```bash
# Crear directorio de trabajo
mkdir -p ~/kcna-labs/lab10
cd ~/kcna-labs/lab10

# Iniciar (o reutilizar) el clúster de un nodo
minikube start --kubernetes-version=v1.30.0 --driver=docker --cpus=2 --memory=4096

# Verificar que el clúster está activo
minikube status
kubectl get nodes

# Definir tu usuario de Docker Hub (reemplaza por el tuyo)
export DOCKERHUB_USER=<tu-usuario-dockerhub>
echo "Usuario Docker Hub: $DOCKERHUB_USER"
```

**Salida esperada** de `kubectl get nodes`:

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   v1.30.0
```

---

## Paso 1: Crear el Namespace y el Deployment inicial (v1.0.0)

**Objetivo:** Desplegar la versión inicial de la aplicación con 4 réplicas y estrategia RollingUpdate explícita, registrando la causa del cambio.

### Instrucciones

1. Crea el namespace del laboratorio:

```bash
kubectl create namespace kcna-delivery
```

2. Crea el manifiesto del Deployment inicial:

```bash
cat > ~/kcna-labs/lab10/deployment-v1.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-delivery
  labels:
    app: kcna-webapp
  annotations:
    kubernetes.io/change-cause: "Despliegue inicial v1.0.0"
spec:
  replicas: 4
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
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        env:
        - name: APP_ENVIRONMENT
          value: "production"
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
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
EOF
```

> **Nota:** El manifiesto usa `${DOCKERHUB_USER}`. Como se creó con `<< EOF` (sin comillas), la variable se sustituye automáticamente por tu usuario. Verifica el resultado con `grep image ~/kcna-labs/lab10/deployment-v1.yaml`.

3. Aplica el manifiesto y crea también un Service para acceder a la app:

```bash
kubectl apply -f ~/kcna-labs/lab10/deployment-v1.yaml

kubectl expose deployment kcna-webapp \
  --name=kcna-webapp-svc \
  --port=8080 --target-port=8080 \
  --type=NodePort \
  -n kcna-delivery
```

4. Espera a que el rollout termine:

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-delivery --timeout=120s
```

### Salida Esperada

```
deployment.apps/kcna-webapp created
service/kcna-webapp-svc exposed
deployment "kcna-webapp" successfully rolled out
```

### Verificación

```bash
kubectl get deployment kcna-webapp -n kcna-delivery
```

Debe mostrar `4/4` en la columna READY. Confirma que la app responde:

```bash
SVC_URL=$(minikube service kcna-webapp-svc -n kcna-delivery --url)
curl -s $SVC_URL/ | python3 -m json.tool
```

Debe devolver un JSON con `version: "1.0.0"`.

---

## Paso 2: Actualizar a v2.0.0 con RollingUpdate (actualización exitosa)

**Objetivo:** Actualizar la imagen y la etiqueta de versión, observando cómo RollingUpdate reemplaza los Pods de forma progresiva sin caída del servicio (`maxUnavailable: 0`).

### Instrucciones

1. En una terminal secundaria, deja observando los Pods en tiempo real:

```bash
kubectl get pods -n kcna-delivery -l app=kcna-webapp -w
```

2. En la terminal principal, actualiza la imagen a la versión 2.0.0 (reutilizamos la misma imagen etiquetándola como v2 para el ejercicio):

```bash
# Etiquetar la imagen existente como 2.0.0 y publicarla (o cargarla en minikube)
docker pull docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
docker tag docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0 docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
```

3. Aplica la actualización de imagen registrando la causa del cambio:

```bash
kubectl set image deployment/kcna-webapp \
  kcna-webapp=docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0 \
  -n kcna-delivery

kubectl annotate deployment/kcna-webapp -n kcna-delivery \
  kubernetes.io/change-cause="Actualización a v2.0.0 (RollingUpdate)" --overwrite
```

4. Observa el progreso del rollout:

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-delivery --timeout=120s
```

### Salida Esperada

En la terminal de observación (`-w`) verás cómo se crea 1 Pod nuevo (`maxSurge: 1`) antes de eliminar uno viejo, manteniendo siempre al menos 4 Pods disponibles (`maxUnavailable: 0`):

```
NAME                           READY   STATUS              RESTARTS   AGE
kcna-webapp-old-...            1/1     Running             0          5m
kcna-webapp-new-...            0/1     ContainerCreating   0          1s
kcna-webapp-new-...            1/1     Running             0          8s
kcna-webapp-old-...            1/1     Terminating         0          5m
...
```

Y el rollout finaliza con:

```
deployment "kcna-webapp" successfully rolled out
```

### Verificación

```bash
kubectl rollout history deployment/kcna-webapp -n kcna-delivery
```

Debe mostrar al menos 2 revisiones:

```
REVISION  CHANGE-CAUSE
1         Despliegue inicial v1.0.0
2         Actualización a v2.0.0 (RollingUpdate)
```

Confirma que los Pods corren la imagen 2.0.0:

```bash
kubectl get pods -n kcna-delivery -l app=kcna-webapp \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}' | sort -u
```

Debe mostrar únicamente la imagen `...:2.0.0`. Detén la observación con `Ctrl+C` en la segunda terminal.

---

## Paso 3: Provocar una actualización fallida y ejecutar rollback

**Objetivo:** Actualizar a una imagen inexistente (v9.9.9), observar cómo el rollout se detiene sin afectar los Pods viejos, y revertir a la revisión estable con `kubectl rollout undo`.

### Instrucciones

1. Aplica una actualización a una imagen que no existe:

```bash
kubectl set image deployment/kcna-webapp \
  kcna-webapp=docker.io/${DOCKERHUB_USER}/kcna-webapp:9.9.9 \
  -n kcna-delivery

kubectl annotate deployment/kcna-webapp -n kcna-delivery \
  kubernetes.io/change-cause="Intento fallido v9.9.9 (imagen inexistente)" --overwrite
```

2. Observa que el rollout se queda atascado (no completa):

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-delivery --timeout=40s
```

3. Diagnostica el problema:

```bash
kubectl get pods -n kcna-delivery -l app=kcna-webapp
kubectl describe deployment kcna-webapp -n kcna-delivery | grep -A5 "Conditions"
```

### Salida Esperada

El `rollout status` no completa y termina con timeout:

```
Waiting for deployment "kcna-webapp" rollout to finish: 1 out of 4 new replicas have been updated...
error: timed out waiting for the condition
```

Los Pods muestran el nuevo ReplicaSet con Pods en `ImagePullBackOff`, pero **los Pods v2.0.0 siguen sirviendo tráfico** (gracias a `maxUnavailable: 0`, el Deployment no elimina Pods sanos hasta que los nuevos estén Ready):

```
NAME                           READY   STATUS             RESTARTS   AGE
kcna-webapp-v2-...             1/1     Running            0          10m
kcna-webapp-bad-...            0/1     ImagePullBackOff   0          30s
```

### Verificación

1. Confirma que la app sigue respondiendo pese al despliegue fallido:

```bash
SVC_URL=$(minikube service kcna-webapp-svc -n kcna-delivery --url)
curl -s -o /dev/null -w "HTTP: %{http_code}\n" $SVC_URL/health
```

Debe devolver `HTTP: 200`.

2. Ejecuta el rollback a la revisión anterior (v2.0.0):

```bash
kubectl rollout undo deployment/kcna-webapp -n kcna-delivery
kubectl rollout status deployment/kcna-webapp -n kcna-delivery --timeout=120s
```

3. Verifica el historial y la imagen actual:

```bash
kubectl rollout history deployment/kcna-webapp -n kcna-delivery
kubectl get pods -n kcna-delivery -l app=kcna-webapp \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}' | sort -u
```

Todos los Pods deben volver a la imagen `...:2.0.0` y estar `Running`.

> **Concepto clave:** Con `maxUnavailable: 0`, un despliegue con imagen inválida **no** provoca caída del servicio: Kubernetes mantiene los Pods viejos hasta que los nuevos pasen la readinessProbe. El rollback restaura el ReplicaSet anterior de forma inmediata.

---

## Paso 4: Contrastar estrategia Recreate vs RollingUpdate

**Objetivo:** Desplegar una segunda aplicación con estrategia `Recreate` y observar que, a diferencia de RollingUpdate, elimina todos los Pods viejos antes de crear los nuevos, provocando una breve indisponibilidad.

### Instrucciones

1. Crea un Deployment con estrategia `Recreate`:

```bash
cat > ~/kcna-labs/lab10/deployment-recreate.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp-recreate
  namespace: kcna-delivery
  labels:
    app: kcna-webapp-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: kcna-webapp-recreate
  template:
    metadata:
      labels:
        app: kcna-webapp-recreate
        version: "1.0.0"
    spec:
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
EOF

kubectl apply -f ~/kcna-labs/lab10/deployment-recreate.yaml
kubectl rollout status deployment/kcna-webapp-recreate -n kcna-delivery --timeout=120s
```

2. En una terminal secundaria observa los Pods:

```bash
kubectl get pods -n kcna-delivery -l app=kcna-webapp-recreate -w
```

3. En la terminal principal, actualiza la imagen para disparar el reemplazo:

```bash
kubectl set image deployment/kcna-webapp-recreate \
  kcna-webapp=docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0 \
  -n kcna-delivery
```

### Salida Esperada

Con `Recreate`, verás que **todos** los Pods viejos pasan a `Terminating` antes de que aparezca ningún Pod nuevo (hay un momento con 0 Pods disponibles):

```
kcna-webapp-recreate-old-...   1/1   Terminating   0   2m
kcna-webapp-recreate-old-...   1/1   Terminating   0   2m
kcna-webapp-recreate-old-...   1/1   Terminating   0   2m
kcna-webapp-recreate-new-...   0/1   Pending       0   0s
kcna-webapp-recreate-new-...   0/1   ContainerCreating ...
```

### Verificación

```bash
kubectl rollout status deployment/kcna-webapp-recreate -n kcna-delivery --timeout=120s
```

Detén la observación con `Ctrl+C`. La diferencia clave:

| Estrategia | Disponibilidad durante el update | Uso típico |
|------------|----------------------------------|-----------|
| **RollingUpdate** | Sin caída (Pods se reemplazan gradualmente) | Servicios web, APIs (default) |
| **Recreate** | Hay downtime (todos abajo, luego arriba) | Apps que no toleran dos versiones simultáneas (ej. migraciones incompatibles) |

---

## Paso 5: Configuración externa con ConfigMap y disparo de rollout

**Objetivo:** Externalizar la configuración de la app en un ConfigMap, inyectarla y comprender que un cambio de ConfigMap montado como env **no** dispara un rollout automático; hay que forzarlo.

### Instrucciones

1. Crea un ConfigMap con configuración de la app:

```bash
kubectl create configmap kcna-webapp-config \
  --from-literal=APP_ENVIRONMENT=production \
  --from-literal=LOG_LEVEL=info \
  -n kcna-delivery
```

2. Actualiza el Deployment principal para consumir el ConfigMap con `envFrom`:

```bash
kubectl patch deployment kcna-webapp -n kcna-delivery --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/envFrom","value":[{"configMapRef":{"name":"kcna-webapp-config"}}]}
]'

kubectl rollout status deployment/kcna-webapp -n kcna-delivery --timeout=120s
```

3. Verifica que la variable llegó al Pod:

```bash
POD=$(kubectl get pods -n kcna-delivery -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n kcna-delivery -- env | grep -E "APP_ENVIRONMENT|LOG_LEVEL"
```

4. Modifica el ConfigMap y observa que los Pods **no** se actualizan solos:

```bash
kubectl patch configmap kcna-webapp-config -n kcna-delivery \
  --type merge -p '{"data":{"LOG_LEVEL":"debug"}}'

# Los Pods existentes conservan el valor viejo en sus variables de entorno
kubectl exec $POD -n kcna-delivery -- env | grep LOG_LEVEL
```

5. Fuerza un nuevo rollout para propagar el cambio:

```bash
kubectl rollout restart deployment/kcna-webapp -n kcna-delivery
kubectl rollout status deployment/kcna-webapp -n kcna-delivery --timeout=120s

NEW_POD=$(kubectl get pods -n kcna-delivery -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -n kcna-delivery -- env | grep LOG_LEVEL
```

### Salida Esperada

- Tras el paso 3: `APP_ENVIRONMENT=production` y `LOG_LEVEL=info`.
- Tras el paso 4 (sin rollout): el Pod aún muestra `LOG_LEVEL=info` (el cambio de ConfigMap no se propaga automáticamente a variables de entorno).
- Tras el paso 5 (con `rollout restart`): el nuevo Pod muestra `LOG_LEVEL=debug`.

### Verificación

```bash
kubectl exec $NEW_POD -n kcna-delivery -- env | grep LOG_LEVEL
```

Debe mostrar `LOG_LEVEL=debug`.

> **Concepto clave:** Los ConfigMaps montados como **variables de entorno** se resuelven al crear el Pod y no cambian en caliente. Los montados como **volumen** sí se actualizan en el filesystem, pero la app debe releerlos. Por eso las herramientas de CI/CD y GitOps suelen forzar un rollout (o usar un hash del ConfigMap en las labels) al cambiar configuración.

---

## Paso 6: Kustomize — variantes declarativas sin duplicar YAML

**Objetivo:** Conocer Kustomize (integrado en kubectl) creando una base y un overlay que cambia el número de réplicas y añade un sufijo de nombre, sin copiar el manifiesto completo.

### Instrucciones

1. Crea la estructura base:

```bash
mkdir -p ~/kcna-labs/lab10/kustomize/base ~/kcna-labs/lab10/kustomize/overlays/staging

# Base: el deployment y su kustomization
cat > ~/kcna-labs/lab10/kustomize/base/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kcna-webapp
  template:
    metadata:
      labels:
        app: kcna-webapp
    spec:
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
EOF

cat > ~/kcna-labs/lab10/kustomize/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF
```

2. Crea un overlay de staging que cambia réplicas y añade prefijo:

```bash
cat > ~/kcna-labs/lab10/kustomize/overlays/staging/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: staging-
namespace: kcna-delivery
resources:
  - ../../base
replicas:
  - name: kcna-webapp
    count: 3
commonLabels:
  environment: staging
EOF
```

3. Previsualiza el resultado renderizado por Kustomize (sin aplicar):

```bash
kubectl kustomize ~/kcna-labs/lab10/kustomize/overlays/staging
```

4. Aplica el overlay:

```bash
kubectl apply -k ~/kcna-labs/lab10/kustomize/overlays/staging
kubectl rollout status deployment/staging-kcna-webapp -n kcna-delivery --timeout=120s
```

### Salida Esperada

`kubectl kustomize` muestra un Deployment con nombre `staging-kcna-webapp`, `replicas: 3` y la label `environment: staging`, generado a partir de la base sin duplicar el YAML completo.

### Verificación

```bash
kubectl get deployment staging-kcna-webapp -n kcna-delivery --show-labels
```

Debe mostrar `3/3` réplicas y la label `environment=staging`.

> **Concepto clave:** Kustomize permite mantener una **base** común y **overlays** por entorno (dev/staging/prod) aplicando parches declarativos (réplicas, imágenes, prefijos, labels). Junto con Helm, es una de las herramientas centrales de la entrega declarativa. Argo CD y Flux (GitOps) consumen directamente bases/overlays de Kustomize o charts de Helm desde Git.

---

## Paso 7 (OPCIONAL): GitOps práctico con Argo CD

> **⏱️ Sección opcional (~20–25 min adicionales).** No forma parte del tiempo base del laboratorio (75 min) ni del script de validación. Requiere ~1 GB de RAM libre adicional y conexión a Internet. Realízala si quieres *ver* GitOps en acción; para el examen KCNA basta con el concepto (que Argo CD/Flux sincronizan el estado del clúster con un repositorio Git de forma declarativa y con auto-reconciliación).

**Objetivo:** Instalar Argo CD en el clúster, registrar una aplicación que apunta a un repositorio Git público con manifiestos de ejemplo, y observar cómo Argo CD **sincroniza** el estado deseado (Git) con el estado real (clúster), demostrando el patrón GitOps y su **self-healing**.

### Instrucciones

1. Instala Argo CD en su propio namespace:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que el servidor de Argo CD esté listo (puede tardar 1-3 min)
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=300s
```

2. Registra una **Application** de Argo CD de forma declarativa (GitOps también para la propia definición). Usamos el repositorio oficial de ejemplos de Argo CD (`guestbook`):

```bash
cat > ~/kcna-labs/lab10/argocd-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: kcna-gitops
  syncPolicy:
    automated:
      prune: true       # elimina recursos que ya no están en Git
      selfHeal: true    # revierte cambios manuales que difieran de Git
    syncOptions:
    - CreateNamespace=true
EOF

kubectl apply -f ~/kcna-labs/lab10/argocd-app.yaml
```

3. Observa cómo Argo CD sincroniza la aplicación desde Git hacia el clúster:

```bash
# Estado de sincronización y salud de la Application
kubectl get application guestbook -n argocd \
  -o jsonpath='{.status.sync.status}{"  /  "}{.status.health.status}{"\n"}'

# Esperar unos segundos a la primera sincronización
sleep 30
kubectl get all -n kcna-gitops
```

### Salida Esperada

Tras la sincronización, la Application queda `Synced` / `Healthy` y aparecen los recursos del guestbook (Deployment + Service) en el namespace `kcna-gitops`:

```
Synced  /  Healthy
```

```
NAME                            READY   STATUS    RESTARTS   AGE
pod/guestbook-ui-...            1/1     Running   0          20s

NAME                    TYPE        CLUSTER-IP     PORT(S)   AGE
service/guestbook-ui    ClusterIP   10.96.x.x      80/TCP    20s
```

### Verificación del self-healing (GitOps en acción)

1. Modifica manualmente el número de réplicas en el clúster (una "deriva" respecto a Git):

```bash
kubectl scale deployment guestbook-ui -n kcna-gitops --replicas=5
kubectl get deployment guestbook-ui -n kcna-gitops
```

2. Espera y observa cómo Argo CD **revierte automáticamente** el cambio (self-heal), porque Git define 1 réplica:

```bash
sleep 30
kubectl get deployment guestbook-ui -n kcna-gitops
```

**Salida esperada:** las réplicas vuelven al valor definido en Git (1), demostrando la **auto-reconciliación**: el estado real converge al estado deseado en Git sin intervención manual.

### (Opcional) Acceder a la UI de Argo CD

```bash
# Contraseña inicial del usuario admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Exponer la UI en localhost:8090 (Ctrl+C para detener)
kubectl port-forward svc/argocd-server -n argocd 8090:443
# Abrir https://localhost:8090 (usuario: admin)
```

### Limpieza de la sección opcional

```bash
kubectl delete -f ~/kcna-labs/lab10/argocd-app.yaml --ignore-not-found
kubectl delete namespace kcna-gitops --ignore-not-found
kubectl delete namespace argocd --ignore-not-found
```

> **Concepto clave — GitOps:** Con Argo CD, **Git es la única fuente de verdad**. El operador de Argo CD compara continuamente el estado del clúster con el repositorio y reconcilia las diferencias (`sync`). `selfHeal: true` revierte cambios manuales; `prune: true` elimina lo que se borró de Git. Flux implementa el mismo patrón. Esto es la evolución del `kubectl apply` manual hacia una entrega **declarativa, auditable y automatizada**.

---

## Validación y Pruebas Finales

Ejecuta el siguiente script para confirmar que el laboratorio se completó correctamente:

```bash
cat > ~/kcna-labs/lab10/validate.sh << 'SCRIPT'
#!/bin/bash
echo "=== Validación Final - Lab 10 ==="
PASS=0; FAIL=0
NS=kcna-delivery

check() { if eval "$2"; then echo "✅ $1"; ((PASS++)); else echo "❌ $1"; ((FAIL++)); fi; }

# 1. Deployment principal con 4 réplicas listas
READY=$(kubectl get deployment kcna-webapp -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "Deployment kcna-webapp con 4 réplicas listas (actual: $READY)" "[ \"$READY\" = \"4\" ]"

# 2. Historial de rollout con >= 3 revisiones
REVS=$(kubectl rollout history deployment/kcna-webapp -n $NS 2>/dev/null | grep -c '^[0-9]')
check "Historial de rollout con >=3 revisiones (actual: $REVS)" "[ \"$REVS\" -ge 3 ]"

# 3. Imagen actual es 2.0.0 (tras el rollback)
IMG=$(kubectl get deployment kcna-webapp -n $NS -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
check "Imagen tras rollback es 2.0.0 (actual: $IMG)" "echo \"$IMG\" | grep -q ':2.0.0'"

# 4. Deployment Recreate existe
check "Deployment kcna-webapp-recreate existe" "kubectl get deployment kcna-webapp-recreate -n $NS >/dev/null 2>&1"

# 5. ConfigMap propagado (LOG_LEVEL=debug)
NEW_POD=$(kubectl get pods -n $NS -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
LL=$(kubectl exec $NEW_POD -n $NS -- env 2>/dev/null | grep LOG_LEVEL | cut -d= -f2)
check "ConfigMap propagado LOG_LEVEL=debug (actual: $LL)" "[ \"$LL\" = \"debug\" ]"

# 6. Overlay de Kustomize aplicado (staging-kcna-webapp con 3 réplicas)
SR=$(kubectl get deployment staging-kcna-webapp -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "Overlay Kustomize staging con 3 réplicas (actual: $SR)" "[ \"$SR\" = \"3\" ]"

echo ""
echo "=== Resultado: $PASS/6 pruebas exitosas, $FAIL fallidas ==="
SCRIPT

chmod +x ~/kcna-labs/lab10/validate.sh
bash ~/kcna-labs/lab10/validate.sh
```

**Salida esperada:** las 6 pruebas deben mostrar `✅`.

---

## Solución de Problemas

### Problema 1: El rollout de v2.0.0 queda atascado y los Pods no arrancan

**Síntomas:**
```
Waiting for deployment "kcna-webapp" rollout to finish: 1 old replicas are pending termination...
```
Los Pods nuevos quedan en `ImagePullBackOff` o `ErrImagePull`.

**Causa:** La imagen `...:2.0.0` no está disponible en el registry ni cargada en Minikube (Minikube usa su propio daemon aislado del Docker del host).

**Solución:**

```bash
# Verificar el error exacto
POD=$(kubectl get pods -n kcna-delivery -l app=kcna-webapp -o jsonpath='{.items[-1].metadata.name}')
kubectl describe pod $POD -n kcna-delivery | grep -A5 "Events"

# Cargar la imagen 2.0.0 en Minikube
minikube image load docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0
minikube image list | grep kcna-webapp

# Reintentar el rollout
kubectl rollout restart deployment/kcna-webapp -n kcna-delivery
```

### Problema 2: `kubectl rollout undo` no revierte a la versión esperada

**Síntomas:** Tras el rollback, la imagen no es la que esperabas o el historial muestra revisiones confusas.

**Causa:** `rollout undo` revierte a la **revisión inmediatamente anterior** por defecto. Si has hecho varios cambios, puede no ser la que quieres.

**Solución:**

```bash
# Listar el historial completo con las causas
kubectl rollout history deployment/kcna-webapp -n kcna-delivery

# Inspeccionar una revisión concreta
kubectl rollout history deployment/kcna-webapp -n kcna-delivery --revision=2

# Revertir a una revisión específica (por ejemplo, la 2 = v2.0.0)
kubectl rollout undo deployment/kcna-webapp -n kcna-delivery --to-revision=2
kubectl rollout status deployment/kcna-webapp -n kcna-delivery
```

---

## Limpieza

```bash
# Eliminar todos los recursos del namespace
kubectl delete namespace kcna-delivery

# Si hiciste el Paso 7 opcional (Argo CD / GitOps), limpia también:
kubectl delete namespace kcna-gitops --ignore-not-found
kubectl delete namespace argocd --ignore-not-found

# (Opcional) Eliminar la imagen 2.0.0 local
docker rmi docker.io/${DOCKERHUB_USER}/kcna-webapp:2.0.0 2>/dev/null

# Verificar limpieza
kubectl get ns | grep kcna-delivery || echo "Namespace eliminado correctamente"
```

> **Nota:** Si vas a continuar con el Lab 11 (Debugging con probes), puedes conservar el clúster Minikube. El Lab 11 crea su propio namespace.

---

## Resumen

En este laboratorio has recorrido el ciclo completo de Application Delivery en Kubernetes:

| Actividad | Resultado |
|-----------|-----------|
| Deployment declarativo v1.0.0 | 4 réplicas con RollingUpdate (`maxSurge=1`, `maxUnavailable=0`) |
| Actualización a v2.0.0 | Reemplazo progresivo sin caída de servicio |
| Actualización fallida + rollback | Servicio estable pese al error; `rollout undo` restaura la versión buena |
| Recreate vs RollingUpdate | Comprensión del trade-off disponibilidad vs simplicidad |
| Configuración externa | ConfigMap consumido con `envFrom`; rollout forzado para propagar cambios |
| Kustomize | Base + overlay de staging sin duplicar YAML |

### Conceptos Clave Reforzados

- **RollingUpdate** es la estrategia por defecto y permite despliegues sin downtime cuando `maxUnavailable` es bajo y las probes están bien configuradas.
- **Rollback** (`kubectl rollout undo`) aprovecha el historial de ReplicaSets que Kubernetes conserva (`revisionHistoryLimit`).
- Un cambio de **ConfigMap** montado como variable de entorno no se propaga solo: hay que forzar un rollout.
- **Kustomize** y **Helm** son las herramientas declarativas de empaquetado; **Argo CD** y **Flux** implementan **GitOps** consumiéndolas desde Git.
- La **cadena de suministro segura** (supply chain) exige imágenes firmadas y escaneadas antes de desplegar.

### Recursos Adicionales

- [Deployments — Documentación oficial](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Rolling updates y rollbacks](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- [Kustomize — Declarative Management](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Helm — Package Manager for Kubernetes](https://helm.sh/docs/)
- [Argo CD](https://argo-cd.readthedocs.io/) · [Flux](https://fluxcd.io/) — GitOps
- [CNCF Cloud Native Glossary](https://glossary.cncf.io/es/)

### Próximo Laboratorio

En el **Lab 11 (Práctica 11)** te centrarás en el debugging de aplicaciones cloud native: readiness/liveness probes, logs de aplicación, variables de configuración y Pods temporales de diagnóstico.
