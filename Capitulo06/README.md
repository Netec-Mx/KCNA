# 11 Práctica 6. Exposición y validación de conectividad

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Kubernetes Services (ClusterIP, NodePort), NGINX Ingress Controller 4.9.1, NetworkPolicy, CoreDNS, kubectl, helm |

## Descripción General

En este laboratorio aplicarás los fundamentos del modelo de red de Kubernetes exponiendo una aplicación mediante diferentes tipos de Services, configurando un Ingress Controller para enrutamiento HTTP basado en rutas, verificando la resolución DNS interna del clúster y aplicando una NetworkPolicy que restrinja el tráfico entre namespaces. Trabajarás sobre un clúster Minikube de 2 nodos con Kubernetes v1.29.2, desplegando una aplicación nginx con 3 réplicas y validando cada capa de conectividad de forma progresiva.

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear y validar un Service de tipo ClusterIP para comunicación interna Pod-to-Service dentro del clúster
- [ ] Exponer una aplicación externamente mediante un Service de tipo NodePort y verificar acceso desde el host
- [ ] Desplegar y configurar el NGINX Ingress Controller 4.9.1 para enrutar tráfico HTTP basado en rutas
- [ ] Aplicar una NetworkPolicy básica de tipo ingress que restrinja el acceso entre namespaces y validar su efecto
- [ ] Verificar la resolución DNS interna del clúster utilizando el servicio CoreDNS desde dentro de un Pod

## Prerrequisitos

### Conocimientos Requeridos

- Comprensión de Pods, Deployments y namespaces (labs del batch 1)
- Familiaridad con manifiestos YAML declarativos de Kubernetes
- Conocimiento básico del modelo de red plana de Kubernetes (Lección 6.1)
- Uso básico de `kubectl` (apply, get, describe, exec, delete)

### Acceso y Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Minikube | 1.32.0 | Clúster local multi-nodo |
| kubectl | 1.29.2 | Gestión del clúster |
| Helm | 3.14.2 | Instalación del Ingress Controller |
| Docker Engine | 26.x | Runtime de contenedores |
| curl | 8.5.0+ | Pruebas de conectividad |

## Entorno del Laboratorio

### Preparación del Clúster

```bash
# Crear directorio de trabajo
mkdir -p ~/kcna-labs/lab06
cd ~/kcna-labs/lab06

# Detener cualquier clúster minikube existente
minikube delete --all 2>/dev/null

# Arrancar clúster con 2 nodos y CNI compatible con NetworkPolicy
minikube start \
  --nodes=2 \
  --kubernetes-version=v1.29.2 \
  --cni=calico \
  --memory=4096 \
  --cpus=2 \
  --driver=docker

# Verificar que ambos nodos estén Ready
kubectl get nodes
```

> **Nota importante**: Usamos `--cni=calico` en lugar del CNI por defecto (kindnet/flannel) porque las NetworkPolicies requieren un plugin CNI que las soporte. Calico es la opción más común para este propósito en Minikube.

**Salida esperada:**

```
NAME           STATUS   ROLES           AGE   VERSION
minikube       Ready    control-plane   90s   v1.29.2
minikube-m02   Ready    <none>          60s   v1.29.2
```

### Verificación de Herramientas

```bash
# Confirmar versiones
kubectl version --client --short 2>/dev/null || kubectl version --client
helm version --short
minikube status
```

## Paso a Paso

---

### Paso 1: Crear el Namespace y Desplegar la Aplicación

**Objetivo:** Crear el namespace `lab06` y desplegar un Deployment con 3 réplicas de nginx:1.25.4 que servirá como aplicación de demostración.

**Instrucciones:**

1. Crea el archivo de namespace:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab06
  labels:
    purpose: networking-lab
    kubernetes.io/metadata.name: lab06
EOF
```

2. Crea el manifiesto del Deployment:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab06
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
```

3. Aplica los manifiestos:

```bash
kubectl apply -f ~/kcna-labs/lab06/namespace.yaml
kubectl apply -f ~/kcna-labs/lab06/deployment.yaml
```

4. Espera a que todos los Pods estén en estado Ready:

```bash
kubectl wait --for=condition=Ready pod -l app=web-app -n lab06 --timeout=120s
```

**Salida esperada:**

```
namespace/lab06 created
deployment.apps/web-app created
pod/web-app-xxxxx-xxxxx condition met
pod/web-app-xxxxx-xxxxx condition met
pod/web-app-xxxxx-xxxxx condition met
```

**Verificación:**

```bash
kubectl get pods -n lab06 -o wide
```

Deberás ver 3 Pods con estado `Running` y `1/1` Ready, distribuidos entre los 2 nodos. Cada Pod tendrá una IP única del rango CIDR del clúster:

```
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE
web-app-6d4f8b7c9-abc12   1/1     Running   0          30s   10.244.1.2      minikube
web-app-6d4f8b7c9-def34   1/1     Running   0          30s   10.244.1.3      minikube
web-app-6d4f8b7c9-ghi56   1/1     Running   0          30s   10.244.205.2    minikube-m02
```

---

### Paso 2: Crear un Service de Tipo ClusterIP

**Objetivo:** Crear un Service ClusterIP que agrupe las 3 réplicas bajo una IP virtual estable y verificar la comunicación interna Pod-to-Service.

**Instrucciones:**

1. Crea el manifiesto del Service ClusterIP:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
  namespace: lab06
  labels:
    app: web-app
    type: clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab06/service-clusterip.yaml
```

3. Verifica que el Service se creó correctamente:

```bash
kubectl get svc web-app-clusterip -n lab06
```

**Salida esperada:**

```
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-app-clusterip   ClusterIP   10.96.xxx.xxx   <none>        80/TCP    5s
```

4. Verifica los endpoints asociados (deben ser las IPs de los 3 Pods):

```bash
kubectl get endpoints web-app-clusterip -n lab06
```

**Salida esperada:**

```
NAME                ENDPOINTS                                      AGE
web-app-clusterip   10.244.1.2:80,10.244.1.3:80,10.244.205.2:80   10s
```

5. Prueba la conectividad interna desde un Pod de diagnóstico:

```bash
# Lanzar Pod temporal para pruebas
kubectl run test-client --rm -it --restart=Never \
  --namespace=lab06 \
  --image=busybox:1.36.1 \
  -- wget -qO- http://web-app-clusterip.lab06.svc.cluster.local
```

**Salida esperada:**

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
pod "test-client" deleted
```

**Verificación:**

La respuesta HTML de nginx confirma que:
- El Service ClusterIP enruta correctamente al backend
- La resolución DNS interna funciona (`web-app-clusterip.lab06.svc.cluster.local`)
- kube-proxy distribuye el tráfico entre las réplicas

```bash
# Verificar describe del Service para confirmar selector y endpoints
kubectl describe svc web-app-clusterip -n lab06 | grep -A2 "Endpoints\|Selector"
```

---

### Paso 3: Crear un Service de Tipo NodePort

**Objetivo:** Exponer la aplicación externamente mediante un Service NodePort en el puerto 30080 y verificar el acceso desde el host.

**Instrucciones:**

1. Crea el manifiesto del Service NodePort:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
  namespace: lab06
  labels:
    app: web-app
    type: nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab06/service-nodeport.yaml
```

3. Verifica el Service:

```bash
kubectl get svc web-app-nodeport -n lab06
```

**Salida esperada:**

```
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web-app-nodeport   NodePort   10.96.xxx.xxx   <none>        80:30080/TCP   5s
```

4. Obtén la IP del nodo de Minikube y prueba el acceso:

```bash
# Obtener la URL del servicio via minikube
MINIKUBE_IP=$(minikube ip)
echo "NodePort URL: http://${MINIKUBE_IP}:30080"

# Probar conectividad desde el host
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://${MINIKUBE_IP}:30080
```

**Salida esperada:**

```
NodePort URL: http://192.168.49.2:30080
HTTP Status: 200
```

5. Verifica que la respuesta contiene contenido de nginx:

```bash
curl -s http://${MINIKUBE_IP}:30080 | head -5
```

**Salida esperada:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

**Verificación:**

```bash
# También puedes usar minikube service para obtener la URL directamente
minikube service web-app-nodeport -n lab06 --url
```

El Service NodePort expone la aplicación en el puerto 30080 de **todos los nodos** del clúster. El tráfico que llega a cualquier nodo en ese puerto es redirigido por kube-proxy a uno de los Pods backend.

---

### Paso 4: Verificar Resolución DNS Interna con CoreDNS

**Objetivo:** Verificar que CoreDNS resuelve correctamente los nombres de servicio dentro del clúster, validando el FQDN completo y las formas abreviadas.

**Instrucciones:**

1. Lanza un Pod de diagnóstico interactivo:

```bash
kubectl run dns-test --rm -it --restart=Never \
  --namespace=lab06 \
  --image=busybox:1.36.1 \
  -- sh
```

2. Dentro del Pod, ejecuta las siguientes pruebas DNS:

```sh
# Resolución con FQDN completo
nslookup web-app-clusterip.lab06.svc.cluster.local

# Resolución con nombre corto (mismo namespace)
nslookup web-app-clusterip

# Resolución cross-namespace del servicio de Kubernetes
nslookup kubernetes.default.svc.cluster.local

# Verificar el servidor DNS configurado
cat /etc/resolv.conf

# Salir del Pod
exit
```

**Salida esperada para `nslookup web-app-clusterip.lab06.svc.cluster.local`:**

```
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:      web-app-clusterip.lab06.svc.cluster.local
Address:   10.96.xxx.xxx
```

**Salida esperada para `cat /etc/resolv.conf`:**

```
nameserver 10.96.0.10
search lab06.svc.cluster.local svc.cluster.local cluster.local
ndots:5
```

**Verificación:**

La resolución DNS confirma que:
- CoreDNS está operativo (servidor en `10.96.0.10`)
- El FQDN `web-app-clusterip.lab06.svc.cluster.local` resuelve a la ClusterIP del Service
- La directiva `search` permite usar nombres cortos dentro del mismo namespace
- El parámetro `ndots:5` asegura que nombres con menos de 5 puntos se busquen primero en los dominios de búsqueda

```bash
# Verificar que CoreDNS está corriendo
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

### Paso 5: Desplegar NGINX Ingress Controller con Helm

**Objetivo:** Instalar el NGINX Ingress Controller versión 4.9.1 usando Helm y crear un recurso Ingress para enrutar tráfico basado en rutas.

**Instrucciones:**

1. Añade el repositorio de Helm si no existe y actualiza:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

2. Habilita el addon de ingress en Minikube (alternativa más simple y compatible):

```bash
# Opción A: Usar addon de minikube (recomendado para labs)
minikube addons enable ingress

# Esperar a que el controller esté listo
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

> **Nota:** Si prefieres instalar con Helm directamente para mayor control de versión:

```bash
# Opción B: Instalación con Helm (versión específica 4.9.1)
# helm install ingress-nginx ingress-nginx/ingress-nginx \
#   --namespace ingress-nginx \
#   --create-namespace \
#   --version 4.9.1 \
#   --set controller.service.type=NodePort \
#   --set controller.service.nodePorts.http=30080 \
#   --wait --timeout=180s
```

> Para este lab usaremos la Opción A (addon de minikube) ya que es más estable en entornos locales. Si ya tienes el NodePort en puerto 30080, el Ingress Controller usará un puerto diferente.

3. Verifica que el Ingress Controller está operativo:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Salida esperada:**

```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running   0          60s
```

4. Crea el recurso Ingress para enrutar `/app` a la aplicación:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: lab06
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: web-app-clusterip
            port:
              number: 80
EOF
```

5. Aplica el Ingress:

```bash
kubectl apply -f ~/kcna-labs/lab06/ingress.yaml
```

6. Verifica el recurso Ingress:

```bash
kubectl get ingress -n lab06
kubectl describe ingress web-app-ingress -n lab06
```

**Salida esperada:**

```
NAME              CLASS   HOSTS   ADDRESS        PORTS   AGE
web-app-ingress   nginx   *       192.168.49.2   80      10s
```

7. Prueba el acceso a través del Ingress:

```bash
# Obtener la IP del Ingress Controller
INGRESS_IP=$(minikube ip)

# Probar la ruta /app
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://${INGRESS_IP}/app

# Ver contenido
curl -s http://${INGRESS_IP}/app | head -3
```

**Salida esperada:**

```
HTTP Status: 200
<!DOCTYPE html>
<html>
<head>
```

**Verificación:**

```bash
# Confirmar que el Ingress tiene Address asignada
kubectl get ingress web-app-ingress -n lab06 -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
echo ""

# Verificar que una ruta inexistente devuelve 404
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://${INGRESS_IP}/noexiste
```

La ruta `/noexiste` debería devolver HTTP 404, confirmando que el enrutamiento por path funciona correctamente.

---

### Paso 6: Aplicar una NetworkPolicy para Restringir Tráfico entre Namespaces

**Objetivo:** Crear una NetworkPolicy que bloquee todo tráfico ingress desde otros namespaces hacia los Pods del namespace `lab06`, y validar su efecto con un Pod en un namespace diferente.

**Instrucciones:**

1. Crea un namespace de prueba:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/namespace-test.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab06-test
  labels:
    purpose: network-policy-test
    kubernetes.io/metadata.name: lab06-test
EOF

kubectl apply -f ~/kcna-labs/lab06/namespace-test.yaml
```

2. Primero, verifica que **sin** NetworkPolicy el tráfico cross-namespace funciona:

```bash
# Lanzar Pod en lab06-test e intentar acceder al Service en lab06
kubectl run cross-ns-test --rm -it --restart=Never \
  --namespace=lab06-test \
  --image=busybox:1.36.1 \
  -- wget -qO- --timeout=5 http://web-app-clusterip.lab06.svc.cluster.local
```

**Salida esperada (antes de la NetworkPolicy):**

```
<!DOCTYPE html>
<html>
...
pod "cross-ns-test" deleted
```

3. Ahora crea la NetworkPolicy que bloquea tráfico de otros namespaces:

```bash
cat <<'EOF' > ~/kcna-labs/lab06/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-ns
  namespace: lab06
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
EOF
```

> **Explicación del manifiesto:**
> - `podSelector.matchLabels.app: web-app` — aplica a los Pods de nuestra aplicación
> - `policyTypes: [Ingress]` — controla solo tráfico entrante
> - `ingress.from.podSelector: {}` — permite tráfico **solo** desde Pods del **mismo namespace** (`lab06`). Al no especificar `namespaceSelector`, se restringe implícitamente al namespace donde reside la policy.

4. Aplica la NetworkPolicy:

```bash
kubectl apply -f ~/kcna-labs/lab06/network-policy.yaml
```

5. Verifica que la policy se creó:

```bash
kubectl get networkpolicy -n lab06
kubectl describe networkpolicy deny-from-other-ns -n lab06
```

**Salida esperada:**

```
NAME                  POD-SELECTOR   AGE
deny-from-other-ns    app=web-app    5s
```

6. Prueba que el tráfico desde otro namespace ahora está **bloqueado**:

```bash
kubectl run cross-ns-blocked --rm -it --restart=Never \
  --namespace=lab06-test \
  --image=busybox:1.36.1 \
  -- wget -qO- --timeout=10 http://web-app-clusterip.lab06.svc.cluster.local
```

**Salida esperada (después de la NetworkPolicy):**

```
wget: download timed out
pod "cross-ns-blocked" deleted
```

> El timeout confirma que la NetworkPolicy está bloqueando el tráfico desde `lab06-test`.

7. Verifica que el tráfico **dentro** del mismo namespace sigue funcionando:

```bash
kubectl run same-ns-test --rm -it --restart=Never \
  --namespace=lab06 \
  --image=busybox:1.36.1 \
  -- wget -qO- --timeout=5 http://web-app-clusterip.lab06.svc.cluster.local
```

**Salida esperada:**

```
<!DOCTYPE html>
<html>
...
pod "same-ns-test" deleted
```

**Verificación:**

```bash
# Resumen del estado de la NetworkPolicy
kubectl get networkpolicy deny-from-other-ns -n lab06 -o yaml | grep -A10 "spec:"
```

El resultado confirma el principio de mínimo privilegio: solo los Pods dentro del namespace `lab06` pueden comunicarse con `web-app`.

---

## Validación y Testing

Ejecuta las siguientes verificaciones finales para confirmar que todos los componentes están correctamente configurados:

```bash
echo "=== VALIDACIÓN COMPLETA DEL LAB 06 ==="
echo ""

# 1. Verificar Deployment
echo "--- 1. Deployment web-app (3/3 réplicas) ---"
kubectl get deployment web-app -n lab06 -o wide
echo ""

# 2. Verificar Services
echo "--- 2. Services ---"
kubectl get svc -n lab06
echo ""

# 3. Verificar ClusterIP internamente
echo "--- 3. Test ClusterIP (interno) ---"
kubectl run final-test-clusterip --rm -it --restart=Never \
  --namespace=lab06 \
  --image=busybox:1.36.1 \
  -- wget -qO- --timeout=5 http://web-app-clusterip:80 2>&1 | head -3
echo ""

# 4. Verificar NodePort externamente
echo "--- 4. Test NodePort (externo) ---"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$(minikube ip):30080
echo ""

# 5. Verificar Ingress
echo "--- 5. Test Ingress (/app) ---"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$(minikube ip)/app
echo ""

# 6. Verificar DNS
echo "--- 6. Test DNS ---"
kubectl run final-test-dns --rm -it --restart=Never \
  --namespace=lab06 \
  --image=busybox:1.36.1 \
  -- nslookup web-app-clusterip.lab06.svc.cluster.local 2>&1 | grep -A1 "Name:"
echo ""

# 7. Verificar NetworkPolicy (debe fallar con timeout)
echo "--- 7. Test NetworkPolicy (cross-namespace - debe fallar) ---"
kubectl run final-test-netpol --rm -it --restart=Never \
  --namespace=lab06-test \
  --image=busybox:1.36.1 \
  -- wget -qO- --timeout=5 http://web-app-clusterip.lab06.svc.cluster.local 2>&1 | tail -1
echo ""

echo "=== VALIDACIÓN COMPLETADA ==="
```

**Criterios de éxito:**

| Test | Resultado Esperado |
|------|-------------------|
| Deployment | 3/3 réplicas Ready |
| ClusterIP interno | HTTP 200 con HTML de nginx |
| NodePort externo | HTTP 200 |
| Ingress /app | HTTP 200 |
| DNS FQDN | Resuelve a la ClusterIP |
| NetworkPolicy cross-ns | Timeout (bloqueado) |
| NetworkPolicy same-ns | HTTP 200 (permitido) |

---

## Troubleshooting

### Problema 1: El Pod de diagnóstico no puede resolver nombres DNS

**Síntomas:**
```
nslookup: can't resolve 'web-app-clusterip.lab06.svc.cluster.local'
wget: bad address 'web-app-clusterip.lab06.svc.cluster.local'
```

**Causa:** CoreDNS no está funcionando correctamente o los Pods de CoreDNS no están en estado Ready. Esto puede ocurrir cuando el CNI (Calico) aún no ha terminado de inicializarse en clústeres recién creados, lo que impide que los Pods de CoreDNS tengan conectividad de red.

**Solución:**

```bash
# 1. Verificar estado de CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Si los Pods están en CrashLoopBackOff o Pending, esperar a que Calico esté listo
kubectl wait --for=condition=Ready pod -l k8s-app=calico-node -n kube-system --timeout=120s

# 3. Reiniciar los Pods de CoreDNS si siguen con problemas
kubectl rollout restart deployment coredns -n kube-system

# 4. Esperar a que estén Ready
kubectl wait --for=condition=Ready pod -l k8s-app=kube-dns -n kube-system --timeout=60s

# 5. Reintentar la prueba DNS
kubectl run dns-fix-test --rm -it --restart=Never \
  --namespace=lab06 \
  --image=busybox:1.36.1 \
  -- nslookup web-app-clusterip.lab06.svc.cluster.local
```

---

### Problema 2: La NetworkPolicy no bloquea el tráfico cross-namespace

**Síntomas:**
El Pod en `lab06-test` sigue pudiendo acceder al Service en `lab06` después de aplicar la NetworkPolicy. El `wget` devuelve HTML en lugar de timeout.

**Causa:** El plugin CNI no soporta NetworkPolicies. Esto ocurre si el clúster se inició sin `--cni=calico` (usando el CNI por defecto de Minikube que no implementa NetworkPolicies), o si los Pods de Calico no están completamente operativos.

**Solución:**

```bash
# 1. Verificar qué CNI está activo
kubectl get pods -n kube-system | grep -E "calico|flannel|kindnet"

# 2. Si no hay Pods de Calico, el clúster se creó sin soporte de NetworkPolicy
# Solución: recrear el clúster con el CNI correcto
minikube delete
minikube start \
  --nodes=2 \
  --kubernetes-version=v1.29.2 \
  --cni=calico \
  --memory=4096 \
  --cpus=2 \
  --driver=docker

# 3. Si Calico existe pero no está Ready, esperar
kubectl wait --for=condition=Ready pod -l k8s-app=calico-node \
  -n kube-system --timeout=180s

# 4. Verificar que la NetworkPolicy está aplicada correctamente
kubectl get networkpolicy -n lab06
kubectl describe networkpolicy deny-from-other-ns -n lab06

# 5. Re-aplicar todos los manifiestos del lab
kubectl apply -f ~/kcna-labs/lab06/
```

---

## Limpieza

```bash
# Eliminar todos los recursos creados en este lab
kubectl delete namespace lab06
kubectl delete namespace lab06-test

# Deshabilitar el addon de ingress (opcional, si quieres liberar recursos)
minikube addons disable ingress

# Verificar limpieza
kubectl get ns | grep lab06

# NOTA: NO eliminar el clúster minikube si planeas continuar con el Lab 09
# que referencia los objetos de este lab. En ese caso, vuelve a crear
# el namespace y recursos antes de ese lab.
```

Si deseas eliminar completamente el clúster:

```bash
# Solo si no necesitas el clúster para labs posteriores
minikube delete --all
```

---

## Resumen

En este laboratorio has aplicado los fundamentos del modelo de red de Kubernetes de forma práctica:

| Concepto | Implementación | Validación |
|----------|---------------|------------|
| Red plana (Pod IPs) | Deployment con 3 réplicas en 2 nodos | `kubectl get pods -o wide` muestra IPs únicas |
| Pod-to-Service (ClusterIP) | Service `web-app-clusterip` | wget desde Pod interno exitoso |
| Externo-a-Service (NodePort) | Service `web-app-nodeport:30080` | curl desde host al NodePort |
| Enrutamiento HTTP (Ingress) | Ingress `web-app-ingress` path `/app` | curl a `/app` devuelve 200 |
| DNS interno (CoreDNS) | FQDN `svc.cluster.local` | nslookup resuelve a ClusterIP |
| Segmentación de red (NetworkPolicy) | Policy `deny-from-other-ns` | Cross-ns timeout, same-ns exitoso |

**Conceptos clave reforzados:**
- Los Services abstraen las IPs efímeras de los Pods proporcionando una IP virtual estable
- CoreDNS permite descubrimiento de servicios por nombre dentro del clúster
- Las NetworkPolicies implementan el principio de mínimo privilegio a nivel de red
- El Ingress Controller actúa como proxy reverso L7 para enrutamiento basado en rutas

### Recursos Adicionales

- [Kubernetes Services — Documentación oficial](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress — Documentación oficial](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Network Policies — Documentación oficial](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [NGINX Ingress Controller — Documentación](https://kubernetes.github.io/ingress-nginx/)

---
