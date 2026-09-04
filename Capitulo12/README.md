# 10 Práctica 12. Observabilidad básica con métricas y logs

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | metrics-server, `kubectl top`, logs de aplicación, `kubectl logs`, agregación de logs, señales de salud |

## Descripción General

En este laboratorio aplicarás los fundamentos de **observabilidad cloud native** de forma práctica y ligera, adecuada para un clúster local. Habilitarás el **metrics-server** de Minikube para recolectar métricas de recursos, consultarás el consumo de CPU/memoria de nodos y Pods con `kubectl top`, generarás carga sobre la aplicación para observar cómo cambian las métricas, trabajarás con **logs de aplicación** (consulta puntual, seguimiento en tiempo real, agregación por label y filtrado por tiempo), y comprenderás la diferencia entre **monitoreo** y **observabilidad** y el rol de los **tres pilares** (métricas, logs y trazas). El laboratorio te da la base conceptual y práctica que Prometheus, Grafana y OpenTelemetry escalan en producción.

## Objetivos de Aprendizaje

- [ ] Habilitar el metrics-server y consultar métricas de CPU/memoria de nodos y Pods con `kubectl top`
- [ ] Generar carga sobre la aplicación y observar la variación de las métricas de recursos en tiempo real
- [ ] Consultar y filtrar logs de aplicación con `kubectl logs` (seguimiento, agregación por label, filtrado por tiempo y número de líneas)
- [ ] Distinguir los tres pilares de la observabilidad (métricas, logs, trazas) e identificar qué herramienta CNCF cubre cada uno
- [ ] Diferenciar monitoreo (qué está fallando) de observabilidad (por qué está fallando) usando señales de salud del clúster

## Prerrequisitos

### Conocimientos previos

- Laboratorios 01–03 completados: imagen `[dockerhub-user]/kcna-webapp:1.0.0` en Docker Hub
- Comprensión de Deployments, Services y requests/limits (Capítulos 2, 3 y 4)
- Uso de `kubectl get`, `logs`, `exec` (Capítulos 3 y 9)
- Conceptos de observabilidad: métricas, logs, trazas (Capítulo 12, lecciones 12.1–12.5)

### Acceso requerido

- Clúster Minikube en ejecución con driver Docker
- `kubectl` configurado y apuntando al clúster
- Imagen `[dockerhub-user]/kcna-webapp:1.0.0` accesible (endpoints `/` y `/health`)

## Entorno del Laboratorio

### Software necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.33.1 | Clúster Kubernetes local |
| Kubernetes | 1.30.0 | Orquestador (vía Minikube) |
| kubectl | 1.30.2 | Cliente CLI de Kubernetes |
| metrics-server | addon Minikube | Recolección de métricas de recursos |
| Docker Engine | 26.1.4 | Runtime de contenedores |

> **Nota de versiones y alcance:** Se usa Kubernetes 1.30.0 / kubectl 1.30.2. Este laboratorio usa el **metrics-server** (ligero) en lugar de desplegar el stack completo de Prometheus + Grafana, para mantenerlo dentro de los 45 minutos y del consumo de recursos de un entorno local. Prometheus, Grafana y OpenTelemetry se abordan a nivel conceptual y como recursos para profundizar.

### Preparación del entorno

```bash
mkdir -p ~/kcna-labs/lab12
cd ~/kcna-labs/lab12

minikube start --kubernetes-version=v1.30.0 --driver=docker --cpus=2 --memory=4096
kubectl create namespace kcna-observ

export DOCKERHUB_USER=<tu-usuario-dockerhub>
echo "Usuario Docker Hub: $DOCKERHUB_USER"
```

**Salida esperada:**

```
namespace/kcna-observ created
```

---

## Paso 1: Habilitar el metrics-server

**Objetivo:** Activar el addon metrics-server de Minikube y confirmar que empieza a recolectar métricas del clúster.

### Instrucciones

1. Habilita el addon:

```bash
minikube addons enable metrics-server
```

2. Espera a que el Pod del metrics-server esté `Ready`:

```bash
kubectl wait --for=condition=Ready pod \
  -l k8s-app=metrics-server \
  -n kube-system --timeout=120s
```

3. Verifica que el API de métricas responde (puede tardar ~30–60 s en tener datos):

```bash
sleep 45
kubectl get apiservices | grep metrics
```

### Salida Esperada

```
✅  metrics-server was successfully enabled
```

```
v1beta1.metrics.k8s.io   kube-system/metrics-server   True   ...
```

### Verificación

```bash
kubectl top nodes
```

Debe mostrar el consumo de CPU y memoria del nodo:

```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   180m         9%     1123Mi          28%
```

> Si `kubectl top nodes` devuelve `error: Metrics API not available`, espera un minuto más: el metrics-server necesita un par de ciclos de scraping antes de tener datos.

---

## Paso 2: Desplegar la aplicación con requests/limits

**Objetivo:** Desplegar `kcna-webapp` con requests y limits definidos para poder interpretar las métricas de utilización frente a las reservas.

### Instrucciones

1. Crea el Deployment y el Service:

```bash
cat > ~/kcna-labs/lab12/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-observ
  labels:
    app: kcna-webapp
spec:
  replicas: 3
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
EOF

kubectl apply -f ~/kcna-labs/lab12/deployment.yaml
kubectl expose deployment kcna-webapp \
  --name=kcna-webapp-svc --port=8080 --target-port=8080 --type=NodePort \
  -n kcna-observ

kubectl rollout status deployment/kcna-webapp -n kcna-observ --timeout=120s
```

### Salida Esperada

```
deployment "kcna-webapp" successfully rolled out
```

### Verificación

```bash
kubectl get pods -n kcna-observ -l app=kcna-webapp
```

Debe mostrar 3 Pods `Running` y `1/1 Ready`.

---

## Paso 3: Consultar métricas de Pods con `kubectl top`

**Objetivo:** Observar el consumo de recursos de los Pods de la aplicación y compararlo con los requests configurados.

### Instrucciones

1. Consulta las métricas de los Pods del namespace (espera ~30 s tras el despliegue para tener datos):

```bash
sleep 30
kubectl top pods -n kcna-observ
```

2. Ordena por consumo de CPU:

```bash
kubectl top pods -n kcna-observ --sort-by=cpu
```

3. Consulta métricas por contenedor:

```bash
kubectl top pods -n kcna-observ --containers
```

### Salida Esperada

```
NAME                           CPU(cores)   MEMORY(bytes)
kcna-webapp-xxxxxxxxx-aaaaa    2m           25Mi
kcna-webapp-xxxxxxxxx-bbbbb    2m           24Mi
kcna-webapp-xxxxxxxxx-ccccc    3m           26Mi
```

En reposo, el consumo de CPU es muy bajo (unos pocos millicores) y la memoria ronda los 25 Mi, muy por debajo del límite de 128 Mi.

### Verificación

```bash
# Comparar el uso frente al request (50m CPU / 64Mi memoria)
echo "Requests configurados: cpu=50m, memoria=64Mi. En reposo, el uso debe ser inferior."
kubectl top pods -n kcna-observ --no-headers
```

> **Concepto clave:** `kubectl top` consume la **Metrics API** que alimenta el metrics-server. Estas mismas métricas son las que usa el **Horizontal Pod Autoscaler (HPA)** para escalar. En producción, Prometheus recolecta un conjunto mucho más rico de métricas (no solo CPU/memoria) exponiéndolas en formato `/metrics`.

---

## Paso 4: Generar carga y observar la variación de métricas

**Objetivo:** Enviar tráfico continuo a la aplicación y observar en tiempo real cómo aumenta el consumo de CPU en las métricas.

### Instrucciones

1. Abre una terminal secundaria y observa las métricas en un bucle:

```bash
# Terminal 2: refrescar métricas cada 5 segundos
while true; do
  clear
  echo "=== kubectl top pods (refresco cada 5s) ==="
  kubectl top pods -n kcna-observ 2>/dev/null
  sleep 5
done
```

2. En la terminal principal, genera carga con un Pod que hace peticiones en bucle:

```bash
kubectl run load-generator --rm -it --restart=Never \
  --image=busybox:1.36.1 -n kcna-observ -- /bin/sh -c '
    echo "Generando carga durante 60 segundos...";
    end=$(( $(date +%s) + 60 ));
    while [ $(date +%s) -lt $end ]; do
      wget -q -O /dev/null http://kcna-webapp-svc.kcna-observ.svc.cluster.local:8080/ 2>/dev/null;
    done;
    echo "Carga finalizada.";
  '
```

### Salida Esperada

En la terminal 2, durante la carga, el consumo de CPU de los Pods sube respecto al reposo:

```
=== kubectl top pods (refresco cada 5s) ===
NAME                           CPU(cores)   MEMORY(bytes)
kcna-webapp-xxxxxxxxx-aaaaa    45m          28Mi
kcna-webapp-xxxxxxxxx-bbbbb    38m          27Mi
kcna-webapp-xxxxxxxxx-ccccc    41m          28Mi
```

Al terminar la carga, el CPU vuelve a valores bajos.

### Verificación

Detén el bucle de la terminal 2 con `Ctrl+C`. Has observado el ciclo básico de monitoreo: una señal (CPU) responde a un evento (carga). Esta es la base de las **alertas** y del **autoescalado**.

> **Concepto clave — monitoreo vs observabilidad:**
> - **Monitoreo**: responde a *"¿qué está pasando?"* con señales predefinidas (CPU alta, errores 5xx). Es reactivo y basado en umbrales conocidos.
> - **Observabilidad**: responde a *"¿por qué está pasando?"* correlacionando métricas, logs y trazas para investigar problemas no anticipados.

---

## Paso 5: Trabajar con logs de aplicación

**Objetivo:** Consultar, seguir, agregar y filtrar los logs de la aplicación, que son el segundo pilar de la observabilidad.

### Instrucciones

1. Consulta los logs de un Pod concreto:

```bash
POD=$(kubectl get pods -n kcna-observ -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n kcna-observ --tail=10
```

2. **Agrega** los logs de todos los Pods del Deployment con `-l` y prefijo por Pod:

```bash
kubectl logs -l app=kcna-webapp -n kcna-observ --prefix --tail=5
```

3. Genera tráfico y **sigue** los logs en tiempo real (en una terminal aparte):

```bash
# Terminal 2: seguir logs
kubectl logs -f $POD -n kcna-observ &
LOGS_PID=$!

# Generar unas peticiones
SVC_URL=$(minikube service kcna-webapp-svc -n kcna-observ --url)
for i in $(seq 1 5); do curl -s -o /dev/null $SVC_URL/; curl -s -o /dev/null $SVC_URL/health; done
sleep 3
kill $LOGS_PID 2>/dev/null
```

4. **Filtra por tiempo** (últimos 2 minutos) y por marca temporal:

```bash
kubectl logs $POD -n kcna-observ --since=2m --timestamps=true | tail -10
```

### Salida Esperada

Los logs muestran las peticiones HTTP servidas por Flask con su código de respuesta:

```
10.244.0.1 - - [.../.../...] "GET / HTTP/1.1" 200 -
10.244.0.1 - - [.../.../...] "GET /health HTTP/1.1" 200 -
```

Con `--prefix`, cada línea va precedida del nombre del Pod que la emitió:

```
[pod/kcna-webapp-xxxxxxxxx-aaaaa/kcna-webapp] 10.244.0.1 - - ... "GET / HTTP/1.1" 200 -
```

### Verificación

```bash
# Contar cuántas peticiones GET / con código 200 hay en los últimos 5 minutos
kubectl logs $POD -n kcna-observ --since=5m | grep -c '"GET / HTTP/1.1" 200'
```

Debe devolver un número mayor que 0.

> **Concepto clave:** `kubectl logs` lee los logs de un contenedor desde el nodo. En producción, herramientas como **Fluentd** o **Fluent Bit** (más ligero) se despliegan como un **DaemonSet** (un agente por nodo) que lee los logs de todos los contenedores del nodo, los enriquece con metadatos de Kubernetes (namespace, Pod, labels) y los envía a un backend central (Elasticsearch, Loki, OpenSearch) para búsqueda y retención. Esto es imprescindible porque los logs de un Pod **se pierden cuando el Pod se elimina**. Fluentd es un proyecto **graduado** de la CNCF.

---

## Paso 6: Los tres pilares y las señales de salud del clúster

**Objetivo:** Consolidar los conceptos identificando los tres pilares de la observabilidad y revisando las señales de salud del clúster (eventos y estado de componentes).

### Instrucciones

1. Revisa las **señales de salud** del clúster mediante eventos recientes:

```bash
kubectl get events -n kcna-observ --sort-by='.lastTimestamp' | tail -15
```

2. Consulta el estado de salud del control plane (readyz):

```bash
kubectl get --raw='/readyz?verbose' 2>/dev/null | head -20
```

3. Genera una tabla-resumen de los tres pilares en tu directorio de trabajo:

```bash
cat > ~/kcna-labs/lab12/tres-pilares.md << 'EOF'
# Los tres pilares de la observabilidad (KCNA)

| Pilar    | Pregunta que responde                        | Ejemplo en este lab           | Herramienta CNCF típica       |
|----------|----------------------------------------------|-------------------------------|-------------------------------|
| Métricas | ¿Cuánto? (valores numéricos en el tiempo)    | `kubectl top` (CPU/memoria)   | Prometheus (+ Grafana)        |
| Logs     | ¿Qué pasó? (eventos discretos con contexto)  | `kubectl logs` (peticiones)   | Fluentd / Fluent Bit + Loki   |
| Trazas   | ¿Dónde/por qué? (recorrido de una petición)  | (conceptual en este lab)      | OpenTelemetry + Jaeger        |

## Monitoreo vs Observabilidad
- Monitoreo: detecta problemas conocidos con umbrales (¿qué falla?).
- Observabilidad: investiga problemas desconocidos correlacionando los tres pilares (¿por qué falla?).

## OpenTelemetry (OTel) — el estándar de instrumentación
- OpenTelemetry es un proyecto CNCF que define un **estándar unificado** (APIs, SDKs y el OTel Collector) para generar y exportar los tres tipos de señales: trazas, métricas y logs.
- Reemplaza instrumentaciones propietarias: instrumentas tu app **una vez** con OTel y exportas a cualquier backend (Jaeger para trazas, Prometheus para métricas, Loki para logs).
- Una **traza** sigue una petición a través de múltiples microservicios (spans encadenados), respondiendo "¿en qué servicio se fue el tiempo?".

## Relación con SRE
- SLI (indicador): una métrica concreta (ej. latencia p99, tasa de errores).
- SLO (objetivo): meta sobre un SLI (ej. 99.9% de peticiones < 300 ms).
- Error budget: margen de incumplimiento tolerado del SLO.
EOF

cat ~/kcna-labs/lab12/tres-pilares.md
```

### Salida Esperada

Los eventos muestran el ciclo de vida reciente (creación de Pods, pulls de imagen, scheduling):

```
...   Normal   Scheduled   pod/kcna-webapp-...   Successfully assigned ...
...   Normal   Pulled      pod/kcna-webapp-...   Container image "..." already present on machine
...   Normal   Started     pod/kcna-webapp-...   Started container kcna-webapp
```

Y `/readyz?verbose` muestra los checks del control plane con `[+]`:

```
[+]ping ok
[+]log ok
[+]etcd ok
...
readyz check passed
```

### Verificación

```bash
test -f ~/kcna-labs/lab12/tres-pilares.md && echo "✅ Documento de los tres pilares generado"
```

---

## Paso 7 (OPCIONAL): Prometheus + Grafana en acción

> **⏱️ Sección opcional (~20–30 min adicionales).** No forma parte del tiempo base del laboratorio (45 min) ni del script de validación. El addon de Prometheus/Grafana consume ~1–2 GB de RAM; asegúrate de haber iniciado Minikube con `--memory=4096` o más. Para el examen KCNA basta con el concepto (Prometheus = métricas, Grafana = visualización); esta sección lo hace tangible.

**Objetivo:** Desplegar el stack de Prometheus + Grafana con el addon integrado de Minikube, ver a Prometheus recolectar métricas y a Grafana visualizarlas en dashboards preconstruidos, conectando lo conceptual con la herramienta real.

### Instrucciones

1. Habilita el addon (despliega Prometheus, Grafana y exporters en el namespace `monitoring`):

```bash
minikube addons enable metrics-server
minikube addons enable dashboard    # opcional, dashboard de Kubernetes

# Stack de Prometheus + Grafana
minikube addons enable prometheus 2>/dev/null || echo "Si tu versión de Minikube no trae el addon 'prometheus', usa la alternativa Helm de más abajo."
```

> **Alternativa con Helm** (si el addon `prometheus` no existe en tu versión de Minikube):
> ```bash
> helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
> helm repo update
> helm install kps prometheus-community/kube-prometheus-stack \
>   --namespace monitoring --create-namespace \
>   --set grafana.adminPassword=admin --wait --timeout=10m
> ```

2. Verifica que los componentes están corriendo:

```bash
kubectl get pods -n monitoring 2>/dev/null || kubectl get pods -A | grep -Ei "prometheus|grafana"
```

3. Accede a **Prometheus** y ejecuta una consulta PromQL:

```bash
# Detecta el Service de Prometheus (nombre varía según addon/Helm)
kubectl get svc -n monitoring | grep -i prometheus

# Port-forward (ajusta el nombre del Service si difiere)
kubectl port-forward -n monitoring svc/prometheus-server 9090:80 2>/dev/null &
PF_PROM=$!
sleep 3
echo "Abre http://localhost:9090 y prueba la consulta PromQL:  up"
echo "También:  sum(rate(container_cpu_usage_seconds_total[1m])) by (namespace)"
```

4. Accede a **Grafana** y explora un dashboard:

```bash
# Obtener el Service de Grafana
kubectl get svc -n monitoring | grep -i grafana

# Port-forward a Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80 2>/dev/null || \
kubectl port-forward -n monitoring svc/kps-grafana 3000:80 2>/dev/null &
sleep 3
echo "Abre http://localhost:3000 (usuario: admin / password: admin o el configurado)"
echo "Explora los dashboards preconstruidos de Kubernetes (CPU/memoria por namespace y Pod)."
```

### Salida Esperada

- `kubectl get pods -n monitoring` muestra Pods de `prometheus-...` y `grafana-...` en estado `Running`.
- En Prometheus (`:9090`), la consulta `up` devuelve series con valor `1` para los targets que está scrapeando.
- En Grafana (`:3000`), los dashboards muestran gráficas de consumo de CPU/memoria del clúster alimentadas por Prometheus.

### Verificación

```bash
# Confirmar que Prometheus está scrapeando targets (al menos 1 target 'up')
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus 2>/dev/null \
  || kubectl get pods -A | grep -i prometheus
```

Detén los port-forward con `kill %1 %2` o `Ctrl+C` cuando termines.

### Limpieza de la sección opcional

```bash
# Si usaste el addon de Minikube
minikube addons disable prometheus 2>/dev/null

# Si usaste Helm
helm uninstall kps -n monitoring 2>/dev/null
kubectl delete namespace monitoring --ignore-not-found
```

> **Concepto clave:** **Prometheus** recolecta (scrapes) métricas exponiéndolas las apps en un endpoint `/metrics` y las almacena como series temporales consultables con **PromQL**. **Grafana** no almacena datos: se conecta a Prometheus (u otras fuentes) para **visualizar** dashboards y definir alertas. Juntos son la combinación estándar de observabilidad de métricas en CNCF, la evolución "de producción" del `kubectl top` que usaste en los pasos anteriores.

---

## Validación y Pruebas Finales

```bash
cat > ~/kcna-labs/lab12/validate.sh << 'SCRIPT'
#!/bin/bash
echo "=== Validación Final - Lab 12 ==="
PASS=0; FAIL=0
NS=kcna-observ

check() { if eval "$2"; then echo "✅ $1"; ((PASS++)); else echo "❌ $1"; ((FAIL++)); fi; }

# 1. metrics-server habilitado y con API disponible
check "Metrics API disponible" "kubectl get apiservices 2>/dev/null | grep -q 'v1beta1.metrics.k8s.io.*True'"

# 2. kubectl top nodes funciona
check "kubectl top nodes devuelve datos" "kubectl top nodes >/dev/null 2>&1"

# 3. kubectl top pods del namespace funciona
check "kubectl top pods -n $NS devuelve datos" "kubectl top pods -n $NS >/dev/null 2>&1"

# 4. Deployment kcna-webapp 3/3
READY=$(kubectl get deployment kcna-webapp -n $NS -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
check "Deployment kcna-webapp con 3 réplicas listas (actual: $READY)" "[ \"$READY\" = \"3\" ]"

# 5. Hay logs de peticiones HTTP
POD=$(kubectl get pods -n $NS -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
LINES=$(kubectl logs $POD -n $NS --since=30m 2>/dev/null | grep -c 'GET')
check "Logs de aplicación con peticiones GET (actual: $LINES)" "[ \"$LINES\" -ge 1 ]"

# 6. Documento de los tres pilares generado
check "Documento tres-pilares.md generado" "test -f ~/kcna-labs/lab12/tres-pilares.md"

echo ""
echo "=== Resultado: $PASS/6 pruebas exitosas, $FAIL fallidas ==="
SCRIPT

chmod +x ~/kcna-labs/lab12/validate.sh
bash ~/kcna-labs/lab12/validate.sh
```

**Salida esperada:** las 6 pruebas deben mostrar `✅`. (Si la prueba 5 falla, genera tráfico con `curl $(minikube service kcna-webapp-svc -n kcna-observ --url)/` y reintenta.)

---

## Solución de Problemas

### Problema 1: `kubectl top` devuelve "Metrics API not available"

**Síntomas:**
```
error: Metrics API not available
```

**Causa:** El metrics-server aún no ha recolectado datos, no está habilitado, o (en algunos entornos Minikube) necesita el flag `--kubelet-insecure-tls`.

**Solución:**

```bash
# 1. Confirmar que el addon está habilitado
minikube addons list | grep metrics-server

# 2. Verificar que el Pod está Running
kubectl get pods -n kube-system -l k8s-app=metrics-server

# 3. Esperar a que haya datos (2-3 ciclos de scraping)
sleep 60
kubectl top nodes

# 4. Si sigue fallando, revisar los logs del metrics-server
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=20
# Si aparece un error de TLS del kubelet, reiniciar el addon:
minikube addons disable metrics-server && minikube addons enable metrics-server
```

### Problema 2: `kubectl top pods -n kcna-observ` no muestra los Pods

**Síntomas:** `kubectl top nodes` funciona pero `kubectl top pods` del namespace no lista nada.

**Causa:** Los Pods llevan menos de un ciclo de scraping en ejecución, o el namespace es incorrecto.

**Solución:**

```bash
# Confirmar que los Pods existen y llevan tiempo corriendo
kubectl get pods -n kcna-observ -o wide

# Esperar un ciclo de scraping adicional
sleep 30
kubectl top pods -n kcna-observ
```

---

## Limpieza

```bash
kubectl delete namespace kcna-observ

# (Opcional) Deshabilitar el metrics-server para liberar recursos
# minikube addons disable metrics-server

kubectl get ns | grep kcna-observ || echo "Namespace eliminado correctamente"
```

> **Nota:** Puedes conservar el clúster para el Lab 13 (laboratorio final integrador). El metrics-server puede quedar habilitado, ya que el Lab 13 lo aprovecha.

---

## Resumen

| Actividad | Resultado |
|-----------|-----------|
| Habilitar metrics-server | Métricas de recursos disponibles vía Metrics API |
| `kubectl top` nodes/pods | Consumo de CPU/memoria en tiempo real |
| Generación de carga | Observación de la variación de métricas ante tráfico |
| Logs de aplicación | Consulta, seguimiento, agregación por label y filtrado por tiempo |
| Tres pilares y señales | Métricas, logs, trazas + eventos y readyz del clúster |

### Conceptos Clave Reforzados

- Los **tres pilares** de la observabilidad son **métricas** (Prometheus), **logs** (Fluentd/Fluent Bit) y **trazas** (OpenTelemetry/Jaeger).
- **Monitoreo** (qué falla, umbrales conocidos) ≠ **observabilidad** (por qué falla, investigación correlacionada).
- `kubectl top` usa la **Metrics API** (metrics-server), la misma que alimenta el **HPA**.
- Los logs de un Pod son **efímeros**: se pierden al eliminarlo; por eso se **agregan** a un backend central.
- La observabilidad es la base del **SRE** (SLI/SLO/error budget) y de un **alerting** efectivo.

### Recursos Adicionales

- [Resource Metrics Pipeline (metrics-server)](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Prometheus](https://prometheus.io/docs/introduction/overview/) · [Grafana](https://grafana.com/docs/)
- [Fluentd](https://www.fluentd.org/) · [Fluent Bit](https://fluentbit.io/)
- [OpenTelemetry](https://opentelemetry.io/docs/) · [Jaeger](https://www.jaegertracing.io/)
- [CNCF Observability Landscape](https://landscape.cncf.io/)

### Próximo Laboratorio

En el **Lab 13 (Práctica 13)** realizarás el laboratorio final integrador: un flujo cloud native completo que combina despliegue, configuración, exposición, seguridad, observabilidad y repaso de conceptos KCNA.
