# 9 Práctica 11. Debugging de una aplicación con probes

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Analizar |
| **Tecnologías** | livenessProbe, readinessProbe, startupProbe, `kubectl logs`, `kubectl exec`, `kubectl debug`, Pods efímeros |

## Descripción General

En este laboratorio te centrarás en el **debugging de aplicaciones** (a diferencia del troubleshooting de infraestructura del Capítulo 9). Distinguirás entre un fallo de la aplicación y un fallo del clúster, configurarás y observarás el efecto de las **readiness**, **liveness** y **startup probes**, provocarás fallos controlados (probe apuntando a un endpoint inexistente, variable de configuración faltante, puerto equivocado), y usarás herramientas de diagnóstico como `kubectl logs --previous`, `kubectl exec` y **Pods/contenedores efímeros de depuración**. El objetivo es adquirir un método claro para analizar por qué una aplicación no arranca, no queda `Ready` o se reinicia en bucle.

## Objetivos de Aprendizaje

- [ ] Diferenciar un problema de aplicación (código/configuración) de un problema de infraestructura (clúster/red) a partir de síntomas y logs
- [ ] Configurar readiness, liveness y startup probes y explicar el rol de cada una en el ciclo de vida del Pod
- [ ] Diagnosticar un Pod que nunca queda `Ready` por una readinessProbe mal configurada y corregirla
- [ ] Diagnosticar un Pod en `CrashLoopBackOff` por una livenessProbe demasiado agresiva o un endpoint incorrecto
- [ ] Usar logs de aplicación (`kubectl logs`, `--previous`), inspección de variables de entorno y validación de puertos/endpoints para localizar la causa raíz
- [ ] Depurar con un Pod efímero temporal (netshoot/busybox) y con `kubectl debug` para inspeccionar red y endpoints desde dentro del clúster

## Prerrequisitos

### Conocimientos previos

- Laboratorios 01–03 completados: imagen `[dockerhub-user]/kcna-webapp:1.0.0` en Docker Hub
- Comprensión de Pods, Deployments y Services (Capítulos 2 y 3)
- Metodología de troubleshooting y estados de Pods (Capítulo 9: Pending, CrashLoopBackOff, ImagePullBackOff)
- Diferencia conceptual entre readiness y liveness probes (Capítulo 11, lección 11.4)

### Acceso requerido

- Clúster Minikube en ejecución con driver Docker
- `kubectl` configurado y apuntando al clúster
- Imagen `[dockerhub-user]/kcna-webapp:1.0.0` accesible (endpoints `/` y `/health`)
- Conexión a Internet para descargar imágenes de diagnóstico

## Entorno del Laboratorio

### Software necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.33.1 | Clúster Kubernetes local |
| Kubernetes | 1.30.0 | Orquestador (vía Minikube) |
| kubectl | 1.30.2 | Cliente CLI de Kubernetes |
| Docker Engine | 26.1.4 | Runtime de contenedores |

> **Nota de versiones:** Se usa Kubernetes 1.30.0 / kubectl 1.30.2. El comando `kubectl debug` (contenedores efímeros) está disponible como GA desde Kubernetes 1.25, por lo que funciona en 1.29 y 1.30 sin activar feature gates.

### Preparación del entorno

```bash
mkdir -p ~/kcna-labs/lab11
cd ~/kcna-labs/lab11

minikube start --kubernetes-version=v1.30.0 --driver=docker --cpus=2 --memory=4096
kubectl create namespace kcna-debug

export DOCKERHUB_USER=<tu-usuario-dockerhub>
echo "Usuario Docker Hub: $DOCKERHUB_USER"
```

**Salida esperada:**

```
namespace/kcna-debug created
```

---

## Paso 1: Desplegar una aplicación sana con las tres probes

**Objetivo:** Desplegar `kcna-webapp` con readiness, liveness y startup probes correctamente configuradas y observar el ciclo de vida saludable del Pod.

### Instrucciones

1. Crea el manifiesto base con las tres probes:

```bash
cat > ~/kcna-labs/lab11/deployment-healthy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-healthy
  namespace: kcna-debug
  labels:
    app: webapp-healthy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-healthy
  template:
    metadata:
      labels:
        app: webapp-healthy
    spec:
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        # startupProbe: da tiempo al arranque antes de activar liveness
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 30
          periodSeconds: 2
        # readinessProbe: decide si el Pod recibe tráfico del Service
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
        # livenessProbe: reinicia el contenedor si deja de estar sano
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
          failureThreshold: 3
EOF

kubectl apply -f ~/kcna-labs/lab11/deployment-healthy.yaml
kubectl rollout status deployment/webapp-healthy -n kcna-debug --timeout=120s
```

2. Observa el estado de los Pods y sus probes:

```bash
kubectl get pods -n kcna-debug -l app=webapp-healthy
POD=$(kubectl get pods -n kcna-debug -l app=webapp-healthy -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n kcna-debug | grep -E "Liveness|Readiness|Startup"
```

### Salida Esperada

```
NAME                              READY   STATUS    RESTARTS   AGE
webapp-healthy-xxxxxxxxx-xxxxx    1/1     Running   0          20s
webapp-healthy-xxxxxxxxx-yyyyy    1/1     Running   0          20s
```

La descripción muestra las tres probes con sus parámetros:

```
    Liveness:   http-get http://:8080/health delay=10s timeout=1s period=15s #success=1 #failure=3
    Readiness:  http-get http://:8080/health delay=3s timeout=1s period=5s #success=1 #failure=3
    Startup:    http-get http://:8080/health delay=0s timeout=1s period=2s #success=1 #failure=30
```

### Verificación

```bash
kubectl get deployment webapp-healthy -n kcna-debug
```

Debe mostrar `2/2` READY.

> **Concepto clave:**
> - **startupProbe**: protege apps de arranque lento; hasta que pasa, liveness/readiness quedan en pausa.
> - **readinessProbe**: si falla, el Pod se retira de los Endpoints del Service (no recibe tráfico) pero **no** se reinicia.
> - **livenessProbe**: si falla repetidamente, el kubelet **reinicia** el contenedor.

---

## Paso 2: Diagnosticar readinessProbe mal configurada (Pod nunca queda Ready)

**Objetivo:** Desplegar un Pod cuya readinessProbe apunta a un endpoint inexistente (`/ready`) y diagnosticar por qué queda `0/1` sin reiniciarse.

### Instrucciones

1. Crea el Deployment con la readinessProbe rota:

```bash
cat > ~/kcna-labs/lab11/deployment-readiness-broken.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-readiness-broken
  namespace: kcna-debug
  labels:
    app: webapp-readiness-broken
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-readiness-broken
  template:
    metadata:
      labels:
        app: webapp-readiness-broken
    spec:
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /ready      # FALLO: la app expone /health, no /ready
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
EOF

kubectl apply -f ~/kcna-labs/lab11/deployment-readiness-broken.yaml
sleep 20
```

2. **Observar** el síntoma:

```bash
kubectl get pods -n kcna-debug -l app=webapp-readiness-broken
```

3. **Describir** para leer los eventos de la probe:

```bash
POD=$(kubectl get pods -n kcna-debug -l app=webapp-readiness-broken -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n kcna-debug | grep -A5 "Events"
```

### Salida Esperada

El Pod está `Running` pero **nunca** `Ready` (0/1), y **no** se reinicia (RESTARTS=0):

```
NAME                                       READY   STATUS    RESTARTS   AGE
webapp-readiness-broken-xxxxxxxxx-xxxxx    0/1     Running   0          20s
```

Eventos:

```
Warning  Unhealthy  ...  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
```

### Verificación y corrección

1. Confirma que la app **sí** responde en `/health` pero **no** en `/ready` (validación de endpoints):

```bash
kubectl exec $POD -n kcna-debug -- wget -qO- -S http://localhost:8080/health 2>&1 | head -3
kubectl exec $POD -n kcna-debug -- wget -qO- -S http://localhost:8080/ready 2>&1 | head -3 || echo "-> /ready devuelve 404 (no existe)"
```

2. Corrige la ruta de la probe y reaplica:

```bash
sed -i 's#path: /ready#path: /health#' ~/kcna-labs/lab11/deployment-readiness-broken.yaml
kubectl apply -f ~/kcna-labs/lab11/deployment-readiness-broken.yaml
kubectl rollout status deployment/webapp-readiness-broken -n kcna-debug --timeout=120s
kubectl get pods -n kcna-debug -l app=webapp-readiness-broken
```

El Pod debe pasar a `1/1 Ready`.

> **Análisis:** Una readinessProbe fallida deja el Pod fuera del Service (sin tráfico) pero **no** lo reinicia. El síntoma clave para distinguirlo de un crash es: `Running`, `0/1`, `RESTARTS=0`. Esto es un problema de **aplicación/configuración**, no de infraestructura.

---

## Paso 3: Diagnosticar livenessProbe agresiva (CrashLoopBackOff)

**Objetivo:** Desplegar un Pod con una livenessProbe que apunta a un puerto equivocado, provocando reinicios continuos, y diagnosticar el patrón CrashLoopBackOff.

### Instrucciones

1. Crea el Deployment con la livenessProbe rota (puerto incorrecto):

```bash
cat > ~/kcna-labs/lab11/deployment-liveness-broken.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-liveness-broken
  namespace: kcna-debug
  labels:
    app: webapp-liveness-broken
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-liveness-broken
  template:
    metadata:
      labels:
        app: webapp-liveness-broken
    spec:
      containers:
      - name: kcna-webapp
        image: docker.io/${DOCKERHUB_USER}/kcna-webapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 9090       # FALLO: la app escucha en 8080, no 9090
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 2
EOF

kubectl apply -f ~/kcna-labs/lab11/deployment-liveness-broken.yaml
sleep 40
```

2. **Observar** los reinicios acumulados:

```bash
kubectl get pods -n kcna-debug -l app=webapp-liveness-broken
```

3. **Describir** para confirmar la causa de los reinicios:

```bash
POD=$(kubectl get pods -n kcna-debug -l app=webapp-liveness-broken -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n kcna-debug | grep -A6 "Events"
```

### Salida Esperada

El contador `RESTARTS` crece y el estado alterna entre `Running` y `CrashLoopBackOff`:

```
NAME                                      READY   STATUS             RESTARTS      AGE
webapp-liveness-broken-xxxxxxxxx-xxxxx    0/1     CrashLoopBackOff   3 (10s ago)   40s
```

Eventos:

```
Warning  Unhealthy  ...  kubelet  Liveness probe failed: Get "http://10.x.x.x:9090/health": dial tcp ...: connect: connection refused
Normal   Killing    ...  kubelet  Container kcna-webapp failed liveness probe, will be restarted
```

### Verificación y corrección

1. Revisa los logs (incluyendo el contenedor anterior con `--previous`) para descartar que sea un fallo del código de la app:

```bash
kubectl logs $POD -n kcna-debug --previous 2>/dev/null | tail -10 || echo "(sin logs previos: el proceso arranca bien; el problema es la probe)"
```

Como la app **sí** arranca correctamente, los logs muestran el arranque normal de Flask: el proceso no falla, es la probe la que lo mata. Esto confirma que el problema es la **configuración de la probe**, no el código.

2. Corrige el puerto de la probe y reaplica:

```bash
sed -i 's/port: 9090/port: 8080/' ~/kcna-labs/lab11/deployment-liveness-broken.yaml
kubectl apply -f ~/kcna-labs/lab11/deployment-liveness-broken.yaml
kubectl rollout status deployment/webapp-liveness-broken -n kcna-debug --timeout=120s
kubectl get pods -n kcna-debug -l app=webapp-liveness-broken
```

El Pod debe estabilizarse en `1/1 Running` con reinicios que dejan de crecer.

> **Análisis:** CrashLoopBackOff por liveness se distingue de un crash de código así: si `kubectl logs --previous` muestra un arranque normal seguido de un `SIGTERM/Killing`, el proceso está sano y es la probe (puerto/ruta/timeout) la que lo reinicia. Si los logs muestran un stacktrace o error de arranque, entonces sí es un problema de la aplicación.

---

## Paso 4: Diagnosticar un fallo de configuración (variable de entorno faltante)

**Objetivo:** Desplegar un Pod que requiere una variable de entorno que no está definida y provoca que el proceso salga con error, diferenciándolo de un fallo de probe.

### Instrucciones

1. Crea un Deployment cuyo contenedor exige `REQUIRED_TOKEN`:

```bash
cat > ~/kcna-labs/lab11/deployment-config-broken.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-config-broken
  namespace: kcna-debug
  labels:
    app: webapp-config-broken
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-config-broken
  template:
    metadata:
      labels:
        app: webapp-config-broken
    spec:
      containers:
      - name: startup-check
        image: busybox:1.36.1
        command: ["/bin/sh", "-c"]
        args:
        - |
          if [ -z "$REQUIRED_TOKEN" ]; then
            echo "FATAL: la variable REQUIRED_TOKEN es obligatoria y no está definida" >&2
            exit 2
          fi
          echo "Token recibido, aplicación iniciada."
          sleep 3600
        env:
        - name: APP_NAME
          value: "kcna-webapp"
        # FALLO: falta REQUIRED_TOKEN
EOF

kubectl apply -f ~/kcna-labs/lab11/deployment-config-broken.yaml
sleep 20
```

2. **Inspeccionar logs** (aquí sí hay un error de aplicación):

```bash
POD=$(kubectl get pods -n kcna-debug -l app=webapp-config-broken -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -n kcna-debug
kubectl logs $POD -n kcna-debug --previous
```

### Salida Esperada

```
NAME                                    READY   STATUS             RESTARTS   AGE
webapp-config-broken-xxxxxxxxx-xxxxx    0/1     CrashLoopBackOff   2          20s
```

Logs:

```
FATAL: la variable REQUIRED_TOKEN es obligatoria y no está definida
```

### Verificación y corrección

1. Confirma el exit code que reporta el error de aplicación:

```bash
kubectl describe pod $POD -n kcna-debug | grep -A3 "Last State"
```

Debe mostrar `Reason: Error` y `Exit Code: 2` (código definido por la app, distinto del 137/SIGKILL de una probe).

2. Inyecta la variable faltante y reaplica:

```bash
kubectl set env deployment/webapp-config-broken -n kcna-debug REQUIRED_TOKEN=abc123
kubectl rollout status deployment/webapp-config-broken -n kcna-debug --timeout=120s
POD=$(kubectl get pods -n kcna-debug -l app=webapp-config-broken -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n kcna-debug
```

Debe mostrar `Token recibido, aplicación iniciada.`

> **Análisis:** Este es un CrashLoopBackOff de **aplicación**: los logs muestran un error explícito y un exit code propio (2). Contrasta con el Paso 3, donde el proceso arrancaba bien y era la probe la que lo mataba. La clave del debugging es leer `kubectl logs --previous` y el `Exit Code`.

---

## Paso 5: Debugging con Pod efímero y `kubectl debug`

**Objetivo:** Usar un Pod temporal de diagnóstico y un contenedor efímero (`kubectl debug`) para inspeccionar conectividad, DNS y endpoints desde dentro del clúster sin modificar la aplicación.

### Instrucciones

1. Crea un Service para la app sana del Paso 1:

```bash
kubectl expose deployment webapp-healthy \
  --name=webapp-healthy-svc --port=8080 --target-port=8080 \
  -n kcna-debug
```

2. Lanza un **Pod efímero temporal** (busybox) para validar el Service y el DNS interno:

```bash
kubectl run debug-tmp --rm -it --restart=Never \
  --image=busybox:1.36.1 -n kcna-debug -- sh -c '
    echo "== Resolución DNS del Service ==";
    nslookup webapp-healthy-svc.kcna-debug.svc.cluster.local;
    echo "== Petición al endpoint /health ==";
    wget -qO- http://webapp-healthy-svc.kcna-debug.svc.cluster.local:8080/health;
    echo;
  '
```

### Salida Esperada

```
== Resolución DNS del Service ==
Name:      webapp-healthy-svc.kcna-debug.svc.cluster.local
Address:   10.96.x.x
== Petición al endpoint /health ==
{"status":"healthy"}
pod "debug-tmp" deleted
```

3. Usa `kubectl debug` para adjuntar un **contenedor efímero** a un Pod en ejecución (útil cuando la imagen de la app es `distroless`/`slim` y no tiene shell ni herramientas de red):

```bash
POD=$(kubectl get pods -n kcna-debug -l app=webapp-healthy -o jsonpath='{.items[0].metadata.name}')

kubectl debug -it $POD -n kcna-debug \
  --image=busybox:1.36.1 \
  --target=kcna-webapp -- sh -c '
    echo "== Comprobando el puerto 8080 en localhost del Pod ==";
    wget -qO- http://localhost:8080/health && echo " -> OK";
  '
```

### Salida Esperada

```
== Comprobando el puerto 8080 en localhost del Pod ==
{"status":"healthy"} -> OK
```

### Verificación

```bash
# Confirmar que los endpoints del Service apuntan a los Pods sanos
kubectl get endpoints webapp-healthy-svc -n kcna-debug
```

Debe mostrar 2 direcciones IP:8080 (las de los 2 Pods de `webapp-healthy`).

> **Concepto clave:** El **Pod efímero** (`kubectl run --rm`) valida el servicio "desde fuera" (perspectiva de otro Pod). El **contenedor efímero** (`kubectl debug --target`) comparte los namespaces de red/proceso del contenedor objetivo, permitiendo depurar "desde dentro" del Pod sin reconstruir la imagen ni añadir herramientas al contenedor de producción.

---

## Validación y Pruebas Finales

```bash
cat > ~/kcna-labs/lab11/validate.sh << 'SCRIPT'
#!/bin/bash
echo "=== Validación Final - Lab 11 ==="
PASS=0; FAIL=0
NS=kcna-debug

check() { if eval "$2"; then echo "✅ $1"; ((PASS++)); else echo "❌ $1"; ((FAIL++)); fi; }

# 1. webapp-healthy 2/2 con tres probes
READY=$(kubectl get deployment webapp-healthy -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "webapp-healthy con 2 réplicas listas (actual: $READY)" "[ \"$READY\" = \"2\" ]"

HASSTARTUP=$(kubectl get deployment webapp-healthy -n $NS -o jsonpath='{.spec.template.spec.containers[0].startupProbe.httpGet.path}' 2>/dev/null)
check "webapp-healthy tiene startupProbe (path: $HASSTARTUP)" "[ -n \"$HASSTARTUP\" ]"

# 2. readiness-broken corregido a /health y Ready
RPATH=$(kubectl get deployment webapp-readiness-broken -n $NS -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}' 2>/dev/null)
RREADY=$(kubectl get deployment webapp-readiness-broken -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "readinessProbe corregida a /health y Ready (path: $RPATH, ready: $RREADY)" "[ \"$RPATH\" = \"/health\" ] && [ \"$RREADY\" = \"1\" ]"

# 3. liveness-broken corregido a puerto 8080 y Ready
LPORT=$(kubectl get deployment webapp-liveness-broken -n $NS -o jsonpath='{.spec.template.spec.containers[0].livenessProbe.httpGet.port}' 2>/dev/null)
LREADY=$(kubectl get deployment webapp-liveness-broken -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "livenessProbe corregida a puerto 8080 y Ready (port: $LPORT, ready: $LREADY)" "[ \"$LPORT\" = \"8080\" ] && [ \"$LREADY\" = \"1\" ]"

# 4. config-broken con REQUIRED_TOKEN definido y Ready
TOK=$(kubectl get deployment webapp-config-broken -n $NS -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="REQUIRED_TOKEN")].value}' 2>/dev/null)
CREADY=$(kubectl get deployment webapp-config-broken -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "REQUIRED_TOKEN inyectado y Ready (token: $TOK, ready: $CREADY)" "[ -n \"$TOK\" ] && [ \"$CREADY\" = \"1\" ]"

# 5. Service webapp-healthy-svc con endpoints
EP=$(kubectl get endpoints webapp-healthy-svc -n $NS -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)
check "Service webapp-healthy-svc con endpoints (ip: $EP)" "[ -n \"$EP\" ]"

echo ""
echo "=== Resultado: $PASS/5 pruebas exitosas, $FAIL fallidas ==="
SCRIPT

chmod +x ~/kcna-labs/lab11/validate.sh
bash ~/kcna-labs/lab11/validate.sh
```

**Salida esperada:** las 5 pruebas deben mostrar `✅`.

---

## Solución de Problemas

### Problema 1: No puedo ver logs con `--previous`

**Síntomas:**
```
error: previous terminated container "..." not found
```

**Causa:** El contenedor aún no se ha reiniciado (no hay instancia previa) o el Pod fue recreado por completo (no reiniciado). `--previous` solo funciona si el **mismo** contenedor ya se reinició al menos una vez.

**Solución:**

```bash
# Ver los logs actuales (sin --previous)
kubectl logs $POD -n kcna-debug

# Ver el motivo del último reinicio en el estado del contenedor
kubectl get pod $POD -n kcna-debug -o jsonpath='{.status.containerStatuses[0].lastState}' | python3 -m json.tool

# Contar reinicios
kubectl get pod $POD -n kcna-debug -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

### Problema 2: `kubectl debug` falla con "ephemeral containers not supported"

**Síntomas:**
```
error: ephemeral containers are disabled for this cluster
```

**Causa:** Versión de Kubernetes anterior a 1.25 o feature gate deshabilitado. En Minikube 1.33.1 con Kubernetes 1.30.0 está habilitado por defecto.

**Solución:**

```bash
# Verificar la versión del servidor
kubectl version -o json | python3 -c "import sys,json; print(json.load(sys.stdin)['serverVersion']['gitVersion'])"

# Si es <1.25, usa un Pod efímero independiente en su lugar
kubectl run debug-tmp --rm -it --restart=Never \
  --image=busybox:1.36.1 -n kcna-debug -- sh
```

---

## Limpieza

```bash
kubectl delete namespace kcna-debug
kubectl get ns | grep kcna-debug || echo "Namespace eliminado correctamente"
```

> **Nota:** Puedes conservar el clúster Minikube para el Lab 12 (Observabilidad), que crea su propio namespace.

---

## Resumen

| Escenario | Síntoma | Herramienta clave | Causa raíz | Corrección |
|-----------|---------|-------------------|-----------|-----------|
| App sana | 1/1 Ready | `describe` (probes) | — | — |
| readinessProbe rota | Running, 0/1, RESTARTS=0 | `describe` (Events 404) | ruta `/ready` inexistente | apuntar a `/health` |
| livenessProbe rota | CrashLoopBackOff, RESTARTS↑ | `logs --previous` (arranque OK) | puerto 9090 incorrecto | apuntar a 8080 |
| Config faltante | CrashLoopBackOff, Exit 2 | `logs --previous` (error explícito) | falta `REQUIRED_TOKEN` | inyectar la variable |
| Diagnóstico de red | — | Pod efímero / `kubectl debug` | — | validar DNS/endpoints |

### Conceptos Clave Reforzados

- **readiness ≠ liveness**: readiness controla el **tráfico** (Endpoints del Service); liveness controla el **reinicio** del contenedor; startup protege el **arranque** lento.
- El síntoma diferencia el tipo de fallo: `Running 0/1 sin reinicios` → readiness; `CrashLoopBackOff con arranque OK en logs` → liveness/probe; `CrashLoopBackOff con error en logs y exit code propio` → aplicación/config.
- `kubectl logs --previous`, el `Exit Code` y el `lastState` son la base del debugging de aplicación.
- Los **Pods/contenedores efímeros** permiten depurar sin alterar la imagen de producción (esencial con imágenes `distroless`).
- Debugging de aplicación (código/config/probes) ≠ troubleshooting de infraestructura (nodos, CNI, storage, RBAC).

### Recursos Adicionales

- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Debug Running Pods (ephemeral containers)](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [kubectl debug — Referencia](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)

### Próximo Laboratorio

En el **Lab 12 (Práctica 12)** trabajarás observabilidad básica: métricas con el metrics-server/Prometheus y consulta de logs, distinguiendo monitoreo de observabilidad.
