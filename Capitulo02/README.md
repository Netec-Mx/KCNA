# 13 Práctica 2. Primer despliegue en Kubernetes

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Prerrequisito** | Lab 01-00-01 completado |

## Descripción General

En este laboratorio iniciarás un clúster Kubernetes local con Minikube, desplegarás la aplicación `kcna-webapp:1.0.0` construida en el laboratorio anterior utilizando manifiestos YAML declarativos, y expondrás el servicio mediante un NodePort para verificar conectividad HTTP desde el host. Aplicarás el flujo completo del modelo declarativo de Kubernetes: escribir YAML → `kubectl apply` → verificar estado.

## Objetivos de Aprendizaje

- [ ] Iniciar un clúster Kubernetes local con Minikube y verificar los componentes del control plane y worker node
- [ ] Crear un Namespace dedicado y desplegar un Deployment con 3 réplicas usando manifiestos YAML declarativos
- [ ] Configurar ConfigMaps y Secrets e inyectarlos como variables de entorno en los Pods
- [ ] Exponer la aplicación con un Service NodePort en el puerto 30080 y verificar conectividad HTTP
- [ ] Aplicar Labels y Annotations para organizar los recursos correctamente

## Prerrequisitos

### Conocimiento Previo

- Laboratorio 01-00-01 completado: imagen `[dockerhub-user]/kcna-webapp:1.0.0` publicada en Docker Hub
- Comprensión básica de YAML (indentación con espacios, listas, mapas clave-valor)
- Familiaridad con comandos básicos de terminal Linux/macOS

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Docker Engine | 26.1.4 | Driver para Minikube |
| Minikube | 1.33.1 | Clúster Kubernetes local |
| kubectl | 1.30.2 | Cliente CLI de Kubernetes |
| curl | 8.8.0 | Verificación de endpoints HTTP |

### Hardware Mínimo

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 4 núcleos | 6+ núcleos |
| RAM | 8 GB | 16 GB |
| Disco | 30 GB libres (SSD) | 50 GB |

## Entorno del Laboratorio

### Preparación del Directorio de Trabajo

```bash
mkdir -p ~/kcna-labs/lab02
cd ~/kcna-labs/lab02
```

### Verificación de Herramientas

```bash
docker --version
minikube version
kubectl version --client
```

**Salida esperada** (versiones aproximadas):

```
Docker version 26.1.4, build ...
minikube version: v1.33.1
Client Version: v1.30.2
```

---

## Paso 1: Iniciar el Clúster Kubernetes con Minikube

### Objetivo

Crear un clúster Kubernetes local v1.30.0 usando Minikube con el driver Docker, y verificar que todos los componentes del control plane estén operativos.

### Instrucciones

1. Inicia el clúster Minikube especificando la versión de Kubernetes y el driver:

```bash
minikube start --kubernetes-version=v1.30.0 --driver=docker --cpus=2 --memory=4096
```

2. Verifica que el clúster esté en estado `Running`:

```bash
minikube status
```

3. Confirma la conectividad de `kubectl` con el clúster:

```bash
kubectl cluster-info
```

4. Lista los nodos del clúster:

```bash
kubectl get nodes -o wide
```

5. Inspecciona los componentes del control plane ejecutándose como Pods en el namespace `kube-system`:

```bash
kubectl get pods -n kube-system
```

### Salida Esperada

Para `minikube status`:

```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Para `kubectl get nodes`:

```
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    ...
minikube   Ready    control-plane   ...   v1.30.0   192.168.49.2   ...
```

Para los Pods del control plane, deberías ver componentes como:

```
coredns-...                  1/1     Running
etcd-minikube                1/1     Running
kube-apiserver-minikube      1/1     Running
kube-controller-manager-...  1/1     Running
kube-proxy-...               1/1     Running
kube-scheduler-minikube      1/1     Running
```

### Verificación

```bash
kubectl get componentstatuses 2>/dev/null || kubectl get --raw='/readyz?verbose' | head -20
```

Confirma que el API server responde correctamente. Si `componentstatuses` muestra deprecación, el endpoint `/readyz` debe indicar `[+]` para los checks principales.

---

## Paso 2: Crear el Namespace y Configurar el Contexto

### Objetivo

Crear el Namespace `kcna-app` donde vivirán todos los recursos de este laboratorio, aplicando Labels y Annotations descriptivas.

### Instrucciones

1. Crea el archivo de manifiesto para el Namespace:

```bash
cat > ~/kcna-labs/lab02/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: kcna-app
  labels:
    project: kcna
    environment: production
    managed-by: kubectl
  annotations:
    description: "Namespace de producción para la aplicación KCNA webapp"
    team: "platform-engineering"
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab02/namespace.yaml
```

3. Verifica la creación del Namespace:

```bash
kubectl get namespace kcna-app --show-labels
```

4. Configura el Namespace como predeterminado para el contexto actual (opcional pero recomendado):

```bash
kubectl config set-context --current --namespace=kcna-app
```

### Salida Esperada

```
namespace/kcna-app created
```

```
NAME       STATUS   AGE   LABELS
kcna-app   Active   ...   environment=production,managed-by=kubectl,project=kcna,...
```

### Verificación

```bash
kubectl config view --minify | grep namespace
```

Debe mostrar `namespace: kcna-app`.

---

## Paso 3: Crear el ConfigMap

### Objetivo

Definir un ConfigMap con la configuración no sensible de la aplicación (variables de entorno `APP_ENV` y `APP_PORT`) que será consumido por el Deployment.

### Instrucciones

1. Crea el manifiesto del ConfigMap:

```bash
cat > ~/kcna-labs/lab02/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: kcna-webapp-config
  namespace: kcna-app
  labels:
    app: kcna-webapp
    project: kcna
    environment: production
  annotations:
    description: "Configuración no sensible para kcna-webapp"
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  APP_VERSION: "1.0.0"
EOF
```

2. Aplica el ConfigMap:

```bash
kubectl apply -f ~/kcna-labs/lab02/configmap.yaml
```

3. Verifica el contenido del ConfigMap:

```bash
kubectl describe configmap kcna-webapp-config -n kcna-app
```

### Salida Esperada

```
configmap/kcna-webapp-config created
```

La salida de `describe` debe mostrar:

```
Name:         kcna-webapp-config
Namespace:    kcna-app
Labels:       app=kcna-webapp
              environment=production
              project=kcna
...
Data
====
APP_ENV:
----
production
APP_PORT:
----
8080
APP_VERSION:
----
1.0.0
```

### Verificación

```bash
kubectl get configmap kcna-webapp-config -n kcna-app -o jsonpath='{.data.APP_ENV}'
```

Debe retornar: `production`

---

## Paso 4: Crear el Secret

### Objetivo

Crear un Secret con datos sensibles (clave secreta de la aplicación) codificados en base64, que será inyectado como variable de entorno en los Pods.

### Instrucciones

1. Genera un valor para la clave secreta:

```bash
APP_SECRET_VALUE=$(openssl rand -base64 32)
echo "Secret generado: $APP_SECRET_VALUE"
```

2. Crea el manifiesto del Secret usando `stringData` (kubectl codifica automáticamente a base64):

```bash
cat > ~/kcna-labs/lab02/secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: kcna-webapp-secret
  namespace: kcna-app
  labels:
    app: kcna-webapp
    project: kcna
    environment: production
  annotations:
    description: "Secretos de la aplicación kcna-webapp"
type: Opaque
stringData:
  APP_SECRET_KEY: "${APP_SECRET_VALUE}"
EOF
```

3. Aplica el Secret:

```bash
kubectl apply -f ~/kcna-labs/lab02/secret.yaml
```

4. Verifica que el Secret existe (los valores se muestran codificados):

```bash
kubectl get secret kcna-webapp-secret -n kcna-app
kubectl describe secret kcna-webapp-secret -n kcna-app
```

### Salida Esperada

```
secret/kcna-webapp-secret created
```

```
NAME                 TYPE     DATA   AGE
kcna-webapp-secret   Opaque   1      ...
```

La salida de `describe` mostrará el tamaño del dato pero **no** su valor:

```
Name:         kcna-webapp-secret
...
Data
====
APP_SECRET_KEY:  44 bytes
```

### Verificación

Para confirmar que el valor se almacenó correctamente (solo en entornos de desarrollo):

```bash
kubectl get secret kcna-webapp-secret -n kcna-app -o jsonpath='{.data.APP_SECRET_KEY}' | base64 --decode
```

Debe mostrar el valor generado en el paso 1.

---

## Paso 5: Crear el Deployment

### Objetivo

Desplegar la aplicación `kcna-webapp:1.0.0` con 3 réplicas, inyectando las variables de entorno desde el ConfigMap y el Secret, con requests/limits de recursos definidos.

### Instrucciones

1. Crea el manifiesto del Deployment:

```bash
cat > ~/kcna-labs/lab02/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-app
  labels:
    app: kcna-webapp
    project: kcna
    environment: production
  annotations:
    description: "Deployment principal de la aplicación KCNA webapp"
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kcna-webapp
  template:
    metadata:
      labels:
        app: kcna-webapp
        version: "1.0.0"
        environment: production
    spec:
      containers:
        - name: kcna-webapp
          image: docker.io/library/kcna-webapp:1.0.0
          ports:
            - containerPort: 8080
              protocol: TCP
              name: http
          envFrom:
            - configMapRef:
                name: kcna-webapp-config
          env:
            - name: APP_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: kcna-webapp-secret
                  key: APP_SECRET_KEY
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
EOF
```

> **Nota importante:** Si publicaste tu imagen en Docker Hub en el lab 01, reemplaza `docker.io/library/kcna-webapp:1.0.0` por `docker.io/[tu-usuario-dockerhub]/kcna-webapp:1.0.0`. Si solo la tienes localmente, primero cárgala en Minikube (ver paso siguiente).

2. **(Si la imagen es local)** Carga la imagen en el entorno de Minikube:

```bash
minikube image load kcna-webapp:1.0.0
```

Si usas la imagen de Docker Hub, actualiza la referencia en el YAML:

```bash
# Reemplaza [TU_USUARIO] con tu usuario de Docker Hub
sed -i "s|docker.io/library/kcna-webapp:1.0.0|docker.io/[TU_USUARIO]/kcna-webapp:1.0.0|" ~/kcna-labs/lab02/deployment.yaml
```

3. Aplica el Deployment:

```bash
kubectl apply -f ~/kcna-labs/lab02/deployment.yaml
```

4. Observa el progreso del despliegue:

```bash
kubectl rollout status deployment/kcna-webapp -n kcna-app --timeout=120s
```

5. Lista los Pods creados:

```bash
kubectl get pods -n kcna-app -l app=kcna-webapp -o wide
```

### Salida Esperada

```
deployment.apps/kcna-webapp created
```

```
deployment "kcna-webapp" successfully rolled out
```

```
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE
kcna-webapp-xxxxxxxxx-xxxxx   1/1     Running   0          ...   10.244.0.x    minikube
kcna-webapp-xxxxxxxxx-xxxxx   1/1     Running   0          ...   10.244.0.x    minikube
kcna-webapp-xxxxxxxxx-xxxxx   1/1     Running   0          ...   10.244.0.x    minikube
```

### Verificación

```bash
kubectl get deployment kcna-webapp -n kcna-app
```

Debe mostrar `3/3` en la columna READY:

```
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
kcna-webapp   3/3     3            3           ...
```

Verifica que las variables de entorno se inyectaron correctamente en un Pod:

```bash
POD_NAME=$(kubectl get pods -n kcna-app -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n kcna-app $POD_NAME -- env | grep -E "APP_ENV|APP_PORT|APP_SECRET_KEY|APP_VERSION"
```

Salida esperada:

```
APP_ENV=production
APP_PORT=8080
APP_VERSION=1.0.0
APP_SECRET_KEY=<valor-generado>
```

---

## Paso 6: Crear el Service NodePort

### Objetivo

Exponer la aplicación fuera del clúster mediante un Service de tipo NodePort en el puerto 30080, permitiendo acceso HTTP desde el host.

### Instrucciones

1. Crea el manifiesto del Service:

```bash
cat > ~/kcna-labs/lab02/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: kcna-webapp-svc
  namespace: kcna-app
  labels:
    app: kcna-webapp
    project: kcna
    environment: production
  annotations:
    description: "Service NodePort para exponer kcna-webapp externamente"
spec:
  type: NodePort
  selector:
    app: kcna-webapp
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30080
EOF
```

2. Aplica el Service:

```bash
kubectl apply -f ~/kcna-labs/lab02/service.yaml
```

3. Verifica el Service:

```bash
kubectl get service kcna-webapp-svc -n kcna-app
```

4. Inspecciona los endpoints asociados al Service (deben ser las IPs de los 3 Pods):

```bash
kubectl get endpoints kcna-webapp-svc -n kcna-app
```

### Salida Esperada

```
service/kcna-webapp-svc created
```

```
NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kcna-webapp-svc   NodePort   10.96.xxx.xxx   <none>        8080:30080/TCP   ...
```

Los endpoints deben mostrar 3 direcciones IP (una por Pod):

```
NAME              ENDPOINTS                                      AGE
kcna-webapp-svc   10.244.0.x:8080,10.244.0.x:8080,10.244.0.x:8080   ...
```

### Verificación

```bash
kubectl describe service kcna-webapp-svc -n kcna-app | grep -A 5 "Endpoints"
```

---

## Paso 7: Verificar Conectividad HTTP

### Objetivo

Acceder a la aplicación desplegada desde el host usando `minikube service` y `curl` para confirmar que el despliegue completo funciona correctamente.

### Instrucciones

1. Obtén la URL del servicio con Minikube:

```bash
minikube service kcna-webapp-svc -n kcna-app --url
```

2. Guarda la URL en una variable y realiza una petición al endpoint principal:

```bash
SERVICE_URL=$(minikube service kcna-webapp-svc -n kcna-app --url)
echo "URL del servicio: $SERVICE_URL"
curl -s $SERVICE_URL | jq .
```

3. Verifica el endpoint de salud:

```bash
curl -s $SERVICE_URL/health | jq .
```

4. Realiza múltiples peticiones para observar el balanceo entre los 3 Pods:

```bash
for i in $(seq 1 6); do
  echo "--- Petición $i ---"
  curl -s $SERVICE_URL | jq -r '.hostname // .host // .'
  sleep 1
done
```

### Salida Esperada

Para el endpoint principal (`GET /`):

```json
{
  "hostname": "kcna-webapp-xxxxxxxxx-xxxxx",
  "version": "1.0.0",
  "environment": "production"
}
```

Para el endpoint de salud (`GET /health`):

```json
{
  "status": "healthy"
}
```

En las peticiones múltiples, deberías observar diferentes hostnames (nombres de Pod), demostrando el balanceo de carga:

```
--- Petición 1 ---
kcna-webapp-xxxxxxxxx-aaaaa
--- Petición 2 ---
kcna-webapp-xxxxxxxxx-bbbbb
--- Petición 3 ---
kcna-webapp-xxxxxxxxx-ccccc
...
```

### Verificación

Confirma el código de respuesta HTTP 200:

```bash
curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/health
```

Debe retornar: `200`

---

## Paso 8: Inspeccionar el Despliegue con kubectl

### Objetivo

Utilizar comandos `kubectl` esenciales (`get`, `describe`, `logs`) para inspeccionar el estado del despliegue y comprender la información que proporciona cada comando.

### Instrucciones

1. Obtén una vista resumida de todos los recursos en el Namespace:

```bash
kubectl get all -n kcna-app
```

2. Describe el Deployment para ver su configuración completa y eventos:

```bash
kubectl describe deployment kcna-webapp -n kcna-app
```

3. Revisa los logs de uno de los Pods:

```bash
POD_NAME=$(kubectl get pods -n kcna-app -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME -n kcna-app --tail=20
```

4. Ejecuta un comando dentro de un Pod para verificar la red interna:

```bash
kubectl exec -n kcna-app $POD_NAME -- wget -qO- http://localhost:8080/health
```

5. Lista los recursos filtrando por Labels:

```bash
kubectl get all -n kcna-app -l project=kcna
kubectl get pods -n kcna-app -l environment=production,app=kcna-webapp
```

### Salida Esperada

El comando `kubectl get all` mostrará:

```
NAME                               READY   STATUS    RESTARTS   AGE
pod/kcna-webapp-xxxxxxxxx-xxxxx   1/1     Running   0          ...
pod/kcna-webapp-xxxxxxxxx-xxxxx   1/1     Running   0          ...
pod/kcna-webapp-xxxxxxxxx-xxxxx   1/1     Running   0          ...

NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kcna-webapp-svc   NodePort   10.96.xxx.xxx   <none>        8080:30080/TCP   ...

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kcna-webapp   3/3     3            3           ...

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/kcna-webapp-xxxxxxxxx   3         3         3       ...
```

### Verificación

Confirma que no hay Pods en estado de error:

```bash
kubectl get pods -n kcna-app --field-selector=status.phase!=Running
```

Si el despliegue es correcto, no debe retornar ningún Pod (o la lista estará vacía).

---

## Validación y Testing Final

Ejecuta esta secuencia completa de validaciones para confirmar que el laboratorio se completó exitosamente:

```bash
echo "=== VALIDACIÓN COMPLETA DEL LAB 02 ==="
echo ""

echo "1. Verificando clúster Minikube..."
minikube status | grep -q "Running" && echo "   ✅ Clúster activo" || echo "   ❌ Clúster no activo"

echo ""
echo "2. Verificando Namespace kcna-app..."
kubectl get ns kcna-app -o jsonpath='{.metadata.labels.environment}' | grep -q "production" && echo "   ✅ Namespace con labels correctos" || echo "   ❌ Labels incorrectos"

echo ""
echo "3. Verificando ConfigMap..."
kubectl get configmap kcna-webapp-config -n kcna-app -o jsonpath='{.data.APP_ENV}' | grep -q "production" && echo "   ✅ ConfigMap correcto" || echo "   ❌ ConfigMap incorrecto"

echo ""
echo "4. Verificando Secret..."
kubectl get secret kcna-webapp-secret -n kcna-app > /dev/null 2>&1 && echo "   ✅ Secret existe" || echo "   ❌ Secret no encontrado"

echo ""
echo "5. Verificando Deployment (3 réplicas)..."
READY=$(kubectl get deployment kcna-webapp -n kcna-app -o jsonpath='{.status.readyReplicas}')
[ "$READY" = "3" ] && echo "   ✅ 3/3 réplicas listas" || echo "   ❌ Réplicas: $READY/3"

echo ""
echo "6. Verificando Service NodePort 30080..."
NODEPORT=$(kubectl get svc kcna-webapp-svc -n kcna-app -o jsonpath='{.spec.ports[0].nodePort}')
[ "$NODEPORT" = "30080" ] && echo "   ✅ NodePort 30080 configurado" || echo "   ❌ NodePort: $NODEPORT"

echo ""
echo "7. Verificando conectividad HTTP..."
SERVICE_URL=$(minikube service kcna-webapp-svc -n kcna-app --url 2>/dev/null)
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/health 2>/dev/null)
[ "$HTTP_CODE" = "200" ] && echo "   ✅ Endpoint /health responde HTTP 200" || echo "   ❌ HTTP Code: $HTTP_CODE"

echo ""
echo "8. Verificando archivos YAML en ~/kcna-labs/lab02/..."
YAML_COUNT=$(ls ~/kcna-labs/lab02/*.yaml 2>/dev/null | wc -l)
[ "$YAML_COUNT" -ge "4" ] && echo "   ✅ $YAML_COUNT manifiestos YAML presentes" || echo "   ❌ Solo $YAML_COUNT archivos YAML"

echo ""
echo "=== FIN DE VALIDACIÓN ==="
```

Todos los checks deben mostrar ✅ para considerar el laboratorio completado.

---

## Solución de Problemas

### Problema 1: Pods en estado ImagePullBackOff o ErrImagePull

**Síntomas:**

```
NAME                           READY   STATUS             RESTARTS   AGE
kcna-webapp-xxxxxxxxx-xxxxx   0/1     ImagePullBackOff   0          2m
```

Al ejecutar `kubectl describe pod`:

```
Warning  Failed   ...  kubelet  Failed to pull image "docker.io/library/kcna-webapp:1.0.0": ...
```

**Causa:** Minikube ejecuta su propio daemon Docker aislado del host. Las imágenes construidas localmente en el Docker del host no están disponibles dentro de Minikube a menos que se carguen explícitamente o se descarguen desde un registry.

**Solución:**

```bash
# Opción A: Cargar la imagen local en Minikube
minikube image load kcna-webapp:1.0.0

# Verificar que la imagen está disponible
minikube image list | grep kcna-webapp

# Opción B: Si la imagen está en Docker Hub, asegúrate de que el YAML
# referencia tu usuario correctamente
# Edita el deployment.yaml y cambia la imagen:
# image: docker.io/[TU_USUARIO]/kcna-webapp:1.0.0

# Reaplicar el Deployment
kubectl apply -f ~/kcna-labs/lab02/deployment.yaml

# Forzar recreación de Pods si es necesario
kubectl rollout restart deployment/kcna-webapp -n kcna-app
```

---

### Problema 2: Service no accesible — curl retorna "Connection refused"

**Síntomas:**

```bash
$ curl $(minikube service kcna-webapp-svc -n kcna-app --url)
curl: (7) Failed to connect to 192.168.49.2 port 30080: Connection refused
```

**Causa:** Los Pods no están en estado `Ready` (la readinessProbe falla), por lo que Kubernetes no los registra como endpoints del Service. Esto suele ocurrir cuando el contenedor no escucha en el puerto 8080 o tarda más de lo esperado en iniciar.

**Solución:**

```bash
# 1. Verificar el estado de los Pods
kubectl get pods -n kcna-app -l app=kcna-webapp

# 2. Verificar los endpoints del Service (si está vacío, los Pods no están Ready)
kubectl get endpoints kcna-webapp-svc -n kcna-app

# 3. Revisar los eventos del Pod para ver por qué falla la probe
POD_NAME=$(kubectl get pods -n kcna-app -l app=kcna-webapp -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME -n kcna-app | grep -A 10 "Events"

# 4. Verificar los logs del contenedor
kubectl logs $POD_NAME -n kcna-app

# 5. Si el contenedor necesita más tiempo para arrancar, ajustar initialDelaySeconds:
# En deployment.yaml, cambiar initialDelaySeconds de 5 a 15 en readinessProbe
# y reaplicar:
kubectl apply -f ~/kcna-labs/lab02/deployment.yaml

# 6. Si el problema es el puerto, verificar que la app escucha en 8080:
kubectl exec -n kcna-app $POD_NAME -- netstat -tlnp 2>/dev/null || \
kubectl exec -n kcna-app $POD_NAME -- ss -tlnp
```

---

## Limpieza

Para conservar los recursos para el laboratorio 03, **no ejecutes la limpieza** a menos que necesites liberar recursos del sistema.

Si necesitas limpiar completamente:

```bash
# Eliminar todos los recursos del Namespace (elimina el Namespace y todo su contenido)
kubectl delete namespace kcna-app

# Detener Minikube (conserva el clúster para reiniciarlo después)
minikube stop

# O eliminar completamente el clúster (requiere recrearlo en el siguiente lab)
# minikube delete
```

Para conservar el entorno para el lab 03 pero solo detener el clúster:

```bash
minikube stop
```

Al reiniciar con `minikube start`, todos los recursos seguirán disponibles.

---

## Resumen

En este laboratorio has completado el flujo declarativo completo de Kubernetes:

| Paso | Recurso | Propósito |
|------|---------|-----------|
| 1 | Clúster Minikube | Infraestructura Kubernetes local |
| 2 | Namespace `kcna-app` | Aislamiento lógico con Labels/Annotations |
| 3 | ConfigMap `kcna-webapp-config` | Configuración no sensible (APP_ENV, APP_PORT) |
| 4 | Secret `kcna-webapp-secret` | Datos sensibles codificados (APP_SECRET_KEY) |
| 5 | Deployment `kcna-webapp` | 3 réplicas con probes y resource limits |
| 6 | Service `kcna-webapp-svc` | Exposición externa vía NodePort 30080 |

**Conceptos clave aplicados:**

- **Modelo declarativo**: describes el estado deseado en YAML y Kubernetes lo mantiene
- **Self-healing**: si un Pod falla, el Deployment lo reemplaza automáticamente
- **Separación de configuración**: ConfigMaps para datos no sensibles, Secrets para datos sensibles
- **Labels y Selectors**: conectan el Service con los Pods correctos
- **Resource requests/limits**: garantizan asignación justa de CPU y memoria

### Archivos Generados

```
~/kcna-labs/lab02/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── deployment.yaml
└── service.yaml
```

### Recursos Adicionales

- [Documentación oficial: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Documentación oficial: Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Documentación oficial: ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Documentación oficial: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Minikube: Handbook](https://minikube.sigs.k8s.io/docs/handbook/)

### Próximo Laboratorio

En el **Lab 03-00-01** implementarás controles de seguridad RBAC sobre estos mismos recursos, creando Roles, ClusterRoles y ServiceAccounts con el principio de mínimo privilegio.
