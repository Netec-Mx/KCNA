# 11 Práctica 3. Administración básica con kubectl

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio dominarás los comandos esenciales de `kubectl` para administrar recursos en un clúster Kubernetes. Partiendo del estado del clúster creado en el lab 02, practicarás inspección detallada de Pods, escalado imperativo y declarativo, actualizaciones de imagen con rollback, gestión de múltiples namespaces y diagnóstico mediante eventos. Al finalizar, tendrás fluidez operativa con `kubectl` y los manifiestos base para el siguiente laboratorio.

## Objetivos de Aprendizaje

- [ ] Utilizar `kubectl get`, `describe`, `logs`, `exec` y `delete` con sus flags más comunes para inspeccionar y gestionar recursos
- [ ] Crear y gestionar múltiples namespaces, cambiando el contexto de trabajo entre ellos
- [ ] Escalar un Deployment de forma imperativa y declarativa, comprendiendo las diferencias entre ambos enfoques
- [ ] Actualizar la imagen de un Deployment y ejecutar un rollback con `kubectl rollout`
- [ ] Diagnosticar problemas leyendo eventos de Kubernetes con `kubectl get events`

## Prerrequisitos

### Conocimientos previos

- Laboratorio 02-00-01 completado: Deployment `kcna-webapp` corriendo en namespace `kcna-app`
- Comprensión básica de manifiestos YAML de Kubernetes (Pod, Deployment, Service)
- Familiaridad con la terminal de Linux/macOS

### Acceso requerido

- Clúster Minikube en ejecución con el Deployment `kcna-webapp` activo
- `kubectl` 1.30.x configurado y apuntando al clúster Minikube
- Directorio de trabajo `~/kcna-labs/` con el contenido del lab 02

## Entorno del Laboratorio

### Software necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.33.1 | Clúster Kubernetes local |
| kubectl | 1.30.2 | CLI de administración |
| Docker Engine | 26.1.4 | Runtime de contenedores |

### Verificación inicial del entorno

```bash
# Confirmar que Minikube está corriendo
minikube status

# Verificar la versión de kubectl
kubectl version --client --short 2>/dev/null || kubectl version --client

# Confirmar el contexto activo
kubectl config current-context
```

**Salida esperada:**

```
minikube
```

```bash
# Verificar que el Deployment del lab anterior existe
kubectl get deployment kcna-webapp -n kcna-app
```

**Salida esperada:**

```
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
kcna-webapp   3/3     3            3           ...
```

### Preparación del directorio de trabajo

```bash
# Crear el directorio del lab 03
mkdir -p ~/kcna-labs/lab03
cd ~/kcna-labs/lab03
```

---

## Paso a Paso

### Paso 1: Inspección detallada de Pods con `kubectl get`

**Objetivo:** Explorar las opciones de formato y filtrado de `kubectl get` para obtener información detallada de los Pods en ejecución.

**Instrucciones:**

1. Lista los Pods del namespace `kcna-app` con información extendida:

```bash
kubectl get pods -n kcna-app -o wide
```

2. Obtén la lista en formato YAML para ver la especificación completa del primer Pod:

```bash
kubectl get pods -n kcna-app -o yaml | head -80
```

3. Usa formato JSON con `jsonpath` para extraer solo los nombres y las IPs de los Pods:

```bash
kubectl get pods -n kcna-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

4. Filtra Pods por etiqueta:

```bash
kubectl get pods -n kcna-app -l app=kcna-webapp
```

5. Lista todos los recursos del namespace:

```bash
kubectl get all -n kcna-app
```

**Salida esperada (ejemplo para el paso 1):**

```
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
kcna-webapp-xxxxxxxxx-xxxxx    1/1     Running   0          10m   172.17.0.3    minikube   <none>           <none>
kcna-webapp-xxxxxxxxx-yyyyy    1/1     Running   0          10m   172.17.0.4    minikube   <none>           <none>
kcna-webapp-xxxxxxxxx-zzzzz    1/1     Running   0          10m   172.17.0.5    minikube   <none>           <none>
```

**Verificación:**

```bash
# Debe mostrar 3 Pods en estado Running
kubectl get pods -n kcna-app --no-headers | wc -l
```

El resultado debe ser `3`.

---

### Paso 2: Diagnóstico con `kubectl describe`

**Objetivo:** Usar `kubectl describe` para leer eventos, condiciones y configuración detallada de un Pod y un Deployment.

**Instrucciones:**

1. Obtén el nombre de un Pod para inspeccionarlo:

```bash
POD_NAME=$(kubectl get pods -n kcna-app -o jsonpath='{.items[0].metadata.name}')
echo "Pod seleccionado: $POD_NAME"
```

2. Describe el Pod completo:

```bash
kubectl describe pod $POD_NAME -n kcna-app
```

3. Observa las secciones clave en la salida:
   - **Labels:** etiquetas asignadas al Pod
   - **Containers:** imagen, puertos, variables de entorno
   - **Conditions:** estado de scheduling, inicialización y readiness
   - **Events:** historial de acciones del kubelet y scheduler

4. Describe el Deployment:

```bash
kubectl describe deployment kcna-webapp -n kcna-app
```

5. Observa en la salida del Deployment:
   - **Replicas:** deseadas, actualizadas, disponibles
   - **Strategy:** tipo de actualización (RollingUpdate)
   - **Events:** escalados y actualizaciones recientes

**Salida esperada (fragmento de eventos del Pod):**

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  12m   default-scheduler  Successfully assigned kcna-app/kcna-webapp-... to minikube
  Normal  Pulled     12m   kubelet            Container image "kcna-webapp:1.0.0" already present on machine
  Normal  Created    12m   kubelet            Created container kcna-webapp
  Normal  Started    12m   kubelet            Started container kcna-webapp
```

**Verificación:**

El comando `describe` debe mostrar `Conditions` con `Ready: True` para el Pod inspeccionado.

---

### Paso 3: Lectura de logs con `kubectl logs`

**Objetivo:** Acceder a los logs de un contenedor en ejecución usando diferentes flags para seguimiento en tiempo real y contenedores reiniciados.

**Instrucciones:**

1. Muestra los logs del Pod seleccionado:

```bash
kubectl logs $POD_NAME -n kcna-app
```

2. Sigue los logs en tiempo real (presiona `Ctrl+C` para salir después de unos segundos):

```bash
kubectl logs $POD_NAME -n kcna-app --follow --tail=10
```

3. Muestra logs de todos los Pods del Deployment usando la etiqueta:

```bash
kubectl logs -l app=kcna-webapp -n kcna-app --prefix
```

4. Genera tráfico para ver logs nuevos (en otra terminal o usando `&`):

```bash
# Obtener la URL del servicio
SVC_URL=$(minikube service kcna-webapp-svc -n kcna-app --url 2>/dev/null)
# Si el servicio es ClusterIP, usa port-forward en background
kubectl port-forward svc/kcna-webapp-svc 8080:8080 -n kcna-app &
PF_PID=$!
sleep 2

# Generar peticiones
curl -s http://localhost:8080/
curl -s http://localhost:8080/health

# Detener port-forward
kill $PF_PID 2>/dev/null
```

5. Revisa los logs nuevamente para confirmar las peticiones registradas:

```bash
kubectl logs $POD_NAME -n kcna-app --tail=5
```

**Salida esperada (ejemplo):**

```
172.17.0.1 - - [01/Jan/2024 10:00:01] "GET / HTTP/1.1" 200 -
172.17.0.1 - - [01/Jan/2024 10:00:02] "GET /health HTTP/1.1" 200 -
```

**Verificación:**

Los logs deben mostrar las peticiones GET realizadas con código de respuesta 200.

---

### Paso 4: Ejecución de comandos dentro del contenedor con `kubectl exec`

**Objetivo:** Usar `kubectl exec` para abrir una sesión interactiva dentro de un contenedor y ejecutar comandos de diagnóstico.

**Instrucciones:**

1. Ejecuta un comando simple sin sesión interactiva:

```bash
kubectl exec $POD_NAME -n kcna-app -- hostname
```

2. Lista las variables de entorno del contenedor:

```bash
kubectl exec $POD_NAME -n kcna-app -- env | sort
```

3. Verifica la conectividad de red desde dentro del contenedor:

```bash
kubectl exec $POD_NAME -n kcna-app -- cat /etc/resolv.conf
```

4. Abre una sesión interactiva dentro del contenedor:

```bash
kubectl exec -it $POD_NAME -n kcna-app -- /bin/sh
```

5. Dentro de la sesión interactiva, ejecuta:

```sh
# Ver procesos corriendo
ps aux
# Ver el sistema de archivos de la app
ls -la /app/
# Verificar conectividad al servicio DNS de Kubernetes
nslookup kubernetes.default.svc.cluster.local 2>/dev/null || cat /etc/resolv.conf
# Salir
exit
```

**Salida esperada (hostname):**

```
kcna-webapp-xxxxxxxxx-xxxxx
```

**Verificación:**

El comando `hostname` dentro del contenedor debe coincidir con el nombre del Pod (`$POD_NAME`).

---

### Paso 5: Escalado imperativo vs declarativo

**Objetivo:** Escalar el Deployment a 5 réplicas de forma imperativa y luego restaurar a 3 réplicas de forma declarativa, comparando ambos enfoques.

**Instrucciones:**

1. **Enfoque imperativo** — Escala a 5 réplicas:

```bash
kubectl scale deployment kcna-webapp -n kcna-app --replicas=5
```

2. Observa el escalado en tiempo real:

```bash
kubectl get pods -n kcna-app -w
```

Presiona `Ctrl+C` cuando los 5 Pods estén en estado `Running`.

3. Confirma el estado del Deployment:

```bash
kubectl get deployment kcna-webapp -n kcna-app
```

**Salida esperada:**

```
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
kcna-webapp   5/5     5            5           ...
```

4. **Enfoque declarativo** — Crea un manifiesto con 3 réplicas:

```bash
cat > ~/kcna-labs/lab03/deployment-3replicas.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-app
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
        image: kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: APP_ENV
          value: "production"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
EOF
```

5. Aplica el manifiesto declarativo:

```bash
kubectl apply -f ~/kcna-labs/lab03/deployment-3replicas.yaml
```

6. Verifica que se redujo a 3 réplicas:

```bash
kubectl get deployment kcna-webapp -n kcna-app
kubectl get pods -n kcna-app
```

**Salida esperada:**

```
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
kcna-webapp   3/3     3            3           ...
```

**Verificación:**

```bash
REPLICAS=$(kubectl get deployment kcna-webapp -n kcna-app -o jsonpath='{.spec.replicas}')
[ "$REPLICAS" -eq 3 ] && echo "✅ Réplicas correctas: $REPLICAS" || echo "❌ Réplicas incorrectas: $REPLICAS"
```

> **Nota sobre imperativo vs declarativo:** El comando `kubectl scale` es rápido pero no queda registrado en ningún archivo. El manifiesto YAML es versionable en Git y representa el estado deseado de forma reproducible. En producción, siempre se prefiere el enfoque declarativo.

---

### Paso 6: Actualización de imagen y rollback

**Objetivo:** Actualizar la imagen del Deployment, revisar el historial de rollout y ejecutar un rollback a la versión anterior.

**Instrucciones:**

1. Revisa el historial de rollout actual:

```bash
kubectl rollout history deployment/kcna-webapp -n kcna-app
```

2. Actualiza la imagen del Deployment imperativamente (simulando un cambio de versión con una variable de entorno diferente):

```bash
kubectl set env deployment/kcna-webapp -n kcna-app APP_VERSION=2.0.0-beta
```

3. Anota el cambio para el historial:

```bash
kubectl annotate deployment/kcna-webapp -n kcna-app kubernetes.io/change-cause="Actualización APP_VERSION a 2.0.0-beta" --overwrite
```

4. Observa el estado del rollout:

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-app
```

**Salida esperada:**

```
deployment "kcna-webapp" successfully rolled out
```

5. Verifica el historial actualizado:

```bash
kubectl rollout history deployment/kcna-webapp -n kcna-app
```

**Salida esperada:**

```
REVISION  CHANGE-CAUSE
1         <none>
2         Actualización APP_VERSION a 2.0.0-beta
```

6. Verifica que los Pods tienen la nueva variable:

```bash
NEW_POD=$(kubectl get pods -n kcna-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -n kcna-app -- env | grep APP_VERSION
```

**Salida esperada:**

```
APP_VERSION=2.0.0-beta
```

7. Ejecuta un rollback a la revisión anterior:

```bash
kubectl rollout undo deployment/kcna-webapp -n kcna-app
```

8. Confirma que el rollback se completó:

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-app
```

9. Verifica que la variable ya no está (o volvió al valor original):

```bash
ROLLED_POD=$(kubectl get pods -n kcna-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $ROLLED_POD -n kcna-app -- env | grep APP_VERSION || echo "APP_VERSION no definida (rollback exitoso)"
```

**Verificación:**

```bash
CURRENT_REV=$(kubectl rollout history deployment/kcna-webapp -n kcna-app | tail -1 | awk '{print $1}')
echo "Revisión actual: $CURRENT_REV"
[ "$CURRENT_REV" -ge 3 ] && echo "✅ Rollback registrado correctamente"
```

---

### Paso 7: Gestión de múltiples namespaces

**Objetivo:** Crear el namespace `kcna-staging`, desplegar una copia de la aplicación y practicar el cambio de contexto entre namespaces.

**Instrucciones:**

1. Crea el namespace `kcna-staging`:

```bash
kubectl create namespace kcna-staging
```

2. Verifica los namespaces existentes:

```bash
kubectl get namespaces
```

3. Crea un manifiesto de Deployment para staging:

```bash
cat > ~/kcna-labs/lab03/deployment-staging.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-staging
  labels:
    app: kcna-webapp
    environment: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kcna-webapp
  template:
    metadata:
      labels:
        app: kcna-webapp
        environment: staging
    spec:
      containers:
      - name: kcna-webapp
        image: kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: APP_ENV
          value: "staging"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
EOF
```

4. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab03/deployment-staging.yaml
```

5. Verifica los Pods en ambos namespaces:

```bash
echo "=== Namespace: kcna-app ==="
kubectl get pods -n kcna-app

echo ""
echo "=== Namespace: kcna-staging ==="
kubectl get pods -n kcna-staging
```

6. Cambia el namespace por defecto del contexto actual para evitar escribir `-n` repetidamente:

```bash
kubectl config set-context --current --namespace=kcna-staging
```

7. Verifica que ahora los comandos apuntan a staging:

```bash
kubectl get pods
# Debe mostrar los Pods de kcna-staging
```

8. Restaura el namespace por defecto:

```bash
kubectl config set-context --current --namespace=default
```

9. Lista Pods en todos los namespaces:

```bash
kubectl get pods --all-namespaces | grep kcna
```

**Salida esperada:**

```
kcna-app       kcna-webapp-xxxxx   1/1     Running   0          ...
kcna-app       kcna-webapp-yyyyy   1/1     Running   0          ...
kcna-app       kcna-webapp-zzzzz   1/1     Running   0          ...
kcna-staging   kcna-webapp-aaaaa   1/1     Running   0          ...
kcna-staging   kcna-webapp-bbbbb   1/1     Running   0          ...
```

**Verificación:**

```bash
STAGING_PODS=$(kubectl get pods -n kcna-staging --no-headers | grep Running | wc -l)
[ "$STAGING_PODS" -eq 2 ] && echo "✅ Staging: $STAGING_PODS Pods corriendo" || echo "❌ Se esperaban 2 Pods en staging"
```

---

### Paso 8: Exploración de API resources y eventos

**Objetivo:** Descubrir los tipos de recursos disponibles en el clúster y diagnosticar el estado del sistema mediante eventos.

**Instrucciones:**

1. Lista todos los recursos API disponibles:

```bash
kubectl api-resources | head -30
```

2. Filtra solo los recursos con namespace (namespaced):

```bash
kubectl api-resources --namespaced=true | grep -E "^NAME|deploy|pod|service|configmap|secret"
```

3. Consulta los eventos del namespace `kcna-app`:

```bash
kubectl get events -n kcna-app --sort-by='.lastTimestamp'
```

4. Filtra eventos de tipo Warning (si existen):

```bash
kubectl get events -n kcna-app --field-selector type=Warning
```

5. Provoca un evento de error para practicar diagnóstico — crea un Pod con imagen inexistente:

```bash
kubectl run pod-error --image=imagen-inexistente:latest -n kcna-app
```

6. Espera 10 segundos y revisa los eventos:

```bash
sleep 10
kubectl get events -n kcna-app --sort-by='.lastTimestamp' | tail -5
```

**Salida esperada (fragmento):**

```
...  Warning   Failed    pod/pod-error   Failed to pull image "imagen-inexistente:latest": ...
...  Warning   BackOff   pod/pod-error   Back-off pulling image "imagen-inexistente:latest"
```

7. Describe el Pod con error para ver el detalle:

```bash
kubectl describe pod pod-error -n kcna-app | grep -A 10 "Events:"
```

8. Elimina el Pod con error:

```bash
kubectl delete pod pod-error -n kcna-app
```

**Verificación:**

```bash
kubectl get pod pod-error -n kcna-app 2>&1 | grep -q "not found" && echo "✅ Pod de error eliminado correctamente"
```

---

### Paso 9: Exploración de kubeconfig y contextos

**Objetivo:** Comprender la estructura del archivo kubeconfig y cómo se gestionan los contextos para conectarse a diferentes clústeres.

**Instrucciones:**

1. Visualiza la configuración actual de kubeconfig:

```bash
kubectl config view
```

2. Lista los contextos disponibles:

```bash
kubectl config get-contexts
```

**Salida esperada:**

```
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default
```

3. Muestra el contexto activo:

```bash
kubectl config current-context
```

4. Examina la ubicación del archivo kubeconfig:

```bash
echo "Archivo kubeconfig: ${KUBECONFIG:-$HOME/.kube/config}"
ls -la ~/.kube/config
```

5. Crea un alias de contexto para el namespace de producción (opcional pero útil):

```bash
kubectl config set-context kcna-prod \
  --cluster=minikube \
  --user=minikube \
  --namespace=kcna-app
```

6. Cambia al nuevo contexto y verifica:

```bash
kubectl config use-context kcna-prod
kubectl get pods
# Debe mostrar los Pods de kcna-app directamente
```

7. Regresa al contexto original:

```bash
kubectl config use-context minikube
kubectl config set-context --current --namespace=default
```

**Verificación:**

```bash
CONTEXT=$(kubectl config current-context)
[ "$CONTEXT" = "minikube" ] && echo "✅ Contexto restaurado: $CONTEXT"
```

---

### Paso 10: Uso de `kubectl delete` y limpieza selectiva

**Objetivo:** Practicar la eliminación selectiva de recursos con diferentes métodos.

**Instrucciones:**

1. Elimina un Pod específico del Deployment en `kcna-app` (el Deployment lo recreará):

```bash
TARGET_POD=$(kubectl get pods -n kcna-app -o jsonpath='{.items[0].metadata.name}')
echo "Eliminando Pod: $TARGET_POD"
kubectl delete pod $TARGET_POD -n kcna-app
```

2. Observa cómo el Deployment recrea el Pod automáticamente:

```bash
kubectl get pods -n kcna-app -w
```

Presiona `Ctrl+C` cuando haya 3 Pods `Running`.

3. Elimina recursos usando el archivo YAML (solo staging para dejar producción intacta):

```bash
# NO ejecutar aún - solo demostración de sintaxis
# kubectl delete -f ~/kcna-labs/lab03/deployment-staging.yaml
echo "Nota: El Deployment de staging se mantiene para el siguiente lab"
```

4. Practica eliminación con etiquetas (dry-run para no eliminar realmente):

```bash
kubectl delete pods -l app=kcna-webapp -n kcna-staging --dry-run=client
```

**Salida esperada:**

```
pod "kcna-webapp-xxxxx" deleted (dry run)
pod "kcna-webapp-yyyyy" deleted (dry run)
```

**Verificación:**

```bash
PROD_PODS=$(kubectl get pods -n kcna-app --no-headers | grep Running | wc -l)
STAGING_PODS=$(kubectl get pods -n kcna-staging --no-headers | grep Running | wc -l)
echo "Producción: $PROD_PODS Pods | Staging: $STAGING_PODS Pods"
[ "$PROD_PODS" -eq 3 ] && [ "$STAGING_PODS" -eq 2 ] && echo "✅ Todos los Pods activos"
```

---

## Validación y Pruebas Finales

Ejecuta este script de validación completo para confirmar que el laboratorio se completó correctamente:

```bash
#!/bin/bash
echo "╔══════════════════════════════════════════════╗"
echo "║  Validación Final - Lab 03-00-01            ║"
echo "╚══════════════════════════════════════════════╝"
echo ""

PASS=0
FAIL=0

# Test 1: Namespace kcna-app existe
if kubectl get namespace kcna-app &>/dev/null; then
  echo "✅ Test 1: Namespace kcna-app existe"
  ((PASS++))
else
  echo "❌ Test 1: Namespace kcna-app no encontrado"
  ((FAIL++))
fi

# Test 2: Namespace kcna-staging existe
if kubectl get namespace kcna-staging &>/dev/null; then
  echo "✅ Test 2: Namespace kcna-staging existe"
  ((PASS++))
else
  echo "❌ Test 2: Namespace kcna-staging no encontrado"
  ((FAIL++))
fi

# Test 3: Deployment en producción con 3 réplicas
PROD_REPLICAS=$(kubectl get deployment kcna-webapp -n kcna-app -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ "$PROD_REPLICAS" -eq 3 ]; then
  echo "✅ Test 3: Deployment producción tiene 3 réplicas"
  ((PASS++))
else
  echo "❌ Test 3: Réplicas en producción = $PROD_REPLICAS (esperado: 3)"
  ((FAIL++))
fi

# Test 4: Deployment en staging con 2 réplicas
STAGING_REPLICAS=$(kubectl get deployment kcna-webapp -n kcna-staging -o jsonpath='{.spec.replicas}' 2>/dev/null)
if [ "$STAGING_REPLICAS" -eq 2 ]; then
  echo "✅ Test 4: Deployment staging tiene 2 réplicas"
  ((PASS++))
else
  echo "❌ Test 4: Réplicas en staging = $STAGING_REPLICAS (esperado: 2)"
  ((FAIL++))
fi

# Test 5: Rollout history tiene al menos 2 revisiones
REVISIONS=$(kubectl rollout history deployment/kcna-webapp -n kcna-app | grep -c "^[0-9]")
if [ "$REVISIONS" -ge 2 ]; then
  echo "✅ Test 5: Historial de rollout tiene $REVISIONS revisiones"
  ((PASS++))
else
  echo "❌ Test 5: Solo $REVISIONS revisión(es) en historial"
  ((FAIL++))
fi

# Test 6: Contexto actual es minikube
CURRENT_CTX=$(kubectl config current-context)
if [ "$CURRENT_CTX" = "minikube" ]; then
  echo "✅ Test 6: Contexto activo es minikube"
  ((PASS++))
else
  echo "❌ Test 6: Contexto activo es $CURRENT_CTX (esperado: minikube)"
  ((FAIL++))
fi

# Test 7: Archivos del lab existen
if [ -f ~/kcna-labs/lab03/deployment-3replicas.yaml ] && [ -f ~/kcna-labs/lab03/deployment-staging.yaml ]; then
  echo "✅ Test 7: Manifiestos YAML del lab03 presentes"
  ((PASS++))
else
  echo "❌ Test 7: Faltan archivos en ~/kcna-labs/lab03/"
  ((FAIL++))
fi

# Test 8: No hay Pods en error
ERROR_PODS=$(kubectl get pods -n kcna-app --no-headers | grep -v Running | wc -l)
if [ "$ERROR_PODS" -eq 0 ]; then
  echo "✅ Test 8: Sin Pods en error en kcna-app"
  ((PASS++))
else
  echo "❌ Test 8: $ERROR_PODS Pod(s) no están Running"
  ((FAIL++))
fi

echo ""
echo "════════════════════════════════════════════════"
echo "  Resultado: $PASS/8 pruebas exitosas, $FAIL fallidas"
echo "════════════════════════════════════════════════"

if [ "$FAIL" -eq 0 ]; then
  echo "  🎉 ¡Laboratorio completado exitosamente!"
else
  echo "  ⚠️  Revisa los tests fallidos antes de continuar"
fi
```

Guarda y ejecuta:

```bash
cat > ~/kcna-labs/lab03/validate.sh << 'SCRIPT'
# (pegar el script anterior aquí)
SCRIPT
chmod +x ~/kcna-labs/lab03/validate.sh
bash ~/kcna-labs/lab03/validate.sh
```

---

## Resolución de Problemas

### Problema 1: Pods quedan en estado `ImagePullBackOff`

**Síntomas:**

```
NAME                           READY   STATUS             RESTARTS   AGE
kcna-webapp-xxxxx-yyyyy        0/1     ImagePullBackOff   0          2m
```

**Causa:** La imagen `kcna-webapp:1.0.0` no está disponible en el registro de imágenes del nodo Minikube. Esto ocurre si la imagen se construyó en Docker local pero no se cargó al entorno de Minikube.

**Solución:**

```bash
# Opción 1: Cargar la imagen al entorno de Minikube
minikube image load kcna-webapp:1.0.0

# Opción 2: Configurar el shell para usar el Docker de Minikube y reconstruir
eval $(minikube docker-env)
cd ~/kcna-labs/lab02
docker build -t kcna-webapp:1.0.0 .

# Verificar que la imagen existe en Minikube
minikube image list | grep kcna-webapp

# Reiniciar el rollout
kubectl rollout restart deployment/kcna-webapp -n kcna-app
```

Asegúrate también de que el manifiesto tenga `imagePullPolicy: IfNotPresent` o `Never` para imágenes locales.

---

### Problema 2: Error `The connection to the server localhost:8080 was refused`

**Síntomas:**

```bash
$ kubectl get pods
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

**Causa:** `kubectl` no puede conectarse al clúster porque Minikube no está corriendo o el archivo kubeconfig no apunta al clúster correcto.

**Solución:**

```bash
# Verificar el estado de Minikube
minikube status

# Si está detenido, iniciarlo
minikube start

# Verificar que el contexto apunta a minikube
kubectl config current-context
# Si no muestra "minikube":
kubectl config use-context minikube

# Verificar la conectividad
kubectl cluster-info

# Si persiste, regenerar la configuración
minikube update-context
```

---

## Limpieza

> **Importante:** NO elimines el Deployment de producción ni el namespace `kcna-staging`. Ambos son necesarios para el lab 04-00-01. Solo limpia los recursos temporales.

```bash
# Eliminar el contexto auxiliar creado (opcional)
kubectl config delete-context kcna-prod 2>/dev/null

# Asegurar que el contexto por defecto está configurado correctamente
kubectl config use-context minikube
kubectl config set-context --current --namespace=default

# Verificar estado final limpio
echo "Estado final del clúster:"
kubectl get deployments --all-namespaces | grep kcna
```

**Estado esperado al finalizar:**

| Namespace | Recurso | Réplicas |
|-----------|---------|----------|
| kcna-app | Deployment/kcna-webapp | 3 |
| kcna-staging | Deployment/kcna-webapp | 2 |

---

## Resumen

### Conceptos clave practicados

| Comando | Propósito | Ejemplo clave |
|---------|-----------|---------------|
| `kubectl get` | Listar recursos | `kubectl get pods -n kcna-app -o wide` |
| `kubectl describe` | Detalle y eventos | `kubectl describe pod <nombre> -n kcna-app` |
| `kubectl logs` | Ver logs del contenedor | `kubectl logs -l app=kcna-webapp --prefix` |
| `kubectl exec` | Ejecutar comandos en contenedor | `kubectl exec -it <pod> -- /bin/sh` |
| `kubectl delete` | Eliminar recursos | `kubectl delete pod <nombre> -n kcna-app` |
| `kubectl scale` | Escalado imperativo | `kubectl scale deployment --replicas=5` |
| `kubectl apply -f` | Aplicación declarativa | `kubectl apply -f deployment.yaml` |
| `kubectl rollout` | Gestión de actualizaciones | `kubectl rollout undo deployment/...` |
| `kubectl config` | Gestión de contextos | `kubectl config set-context --current --namespace=...` |
| `kubectl api-resources` | Descubrir tipos de recursos | `kubectl api-resources --namespaced=true` |
| `kubectl get events` | Diagnóstico del clúster | `kubectl get events --sort-by='.lastTimestamp'` |

### Imperativo vs Declarativo — Resumen

| Aspecto | Imperativo | Declarativo |
|---------|-----------|-------------|
| Velocidad | Rápido para cambios puntuales | Requiere editar archivo |
| Reproducibilidad | Baja (no queda registro) | Alta (versionable en Git) |
| Uso recomendado | Debugging, exploración | Producción, CI/CD |
| Ejemplo | `kubectl scale --replicas=5` | `kubectl apply -f deploy.yaml` |

### Archivos generados

```
~/kcna-labs/lab03/
├── deployment-3replicas.yaml    # Deployment producción (3 réplicas)
├── deployment-staging.yaml      # Deployment staging (2 réplicas)
└── validate.sh                  # Script de validación
```

### Recursos adicionales

- [kubectl Cheat Sheet — Documentación oficial](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Gestión declarativa de objetos Kubernetes](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Debugging Pods — Kubernetes Docs](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [kubectl rollout — Referencia de comandos](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

---
