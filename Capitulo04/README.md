# Scheduling básico y análisis de Pods pendientes

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Minikube 1.33.1 (multi-nodo), Kubernetes 1.30.0, kube-scheduler, Resource Requests/Limits, nodeSelector, Node Affinity, Taints/Tolerations |

## Descripción general

En este laboratorio configurarás un clúster Minikube de 3 nodos para explorar el comportamiento del kube-scheduler. Provocarás Pods en estado `Pending` por recursos insuficientes, dirigirás Pods a nodos específicos con `nodeSelector` y `nodeAffinity`, y controlarás la exclusión de nodos mediante `taints` y `tolerations`. Cada escenario se diagnosticará con `kubectl describe` para interpretar los eventos de scheduling.

## Objetivos de aprendizaje

- [ ] Provocar un Pod en estado `Pending` por recursos insuficientes y diagnosticar la causa leyendo eventos del scheduler
- [ ] Aplicar `nodeSelector` para forzar la ejecución de un Pod en un nodo específico mediante labels personalizadas
- [ ] Configurar un `taint` en un nodo y verificar que los Pods sin `toleration` no se programan en él
- [ ] Añadir una `toleration` a un Deployment para permitir scheduling en un nodo con taint
- [ ] Usar `preferredDuringSchedulingIgnoredDuringExecution` para expresar preferencia de nodo sin restricción estricta

## Prerrequisitos

### Conocimientos previos

- Familiaridad con manifiestos YAML de Kubernetes (Deployments, Pods, Services)
- Manejo de `kubectl apply`, `get`, `describe`, `delete`
- Comprensión básica de CPU (millicores) y memoria (Mi/Gi) en Linux

### Acceso requerido

- Imagen `[dockerhub-user]/kcna-webapp:1.0.0` publicada en Docker Hub (lab 01)
- Minikube 1.33.1 instalado con driver Docker
- Mínimo 6 GB de RAM disponibles para el clúster de 3 nodos
- `kubectl` 1.30.x configurado

## Entorno del laboratorio

### Requisitos de hardware

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 4 núcleos | 6 núcleos |
| RAM | 8 GB | 16 GB |
| Disco | 30 GB libres (SSD) | 50 GB |

### Software utilizado

| Herramienta | Versión |
|-------------|---------|
| Minikube | 1.33.1 |
| Kubernetes | 1.30.0 |
| kubectl | 1.30.2 |
| Docker Engine | 26.1.4 |

### Preparación del entorno

```bash
# Crear directorio de trabajo
mkdir -p ~/kcna-labs/lab04
cd ~/kcna-labs/lab04

# Detener cualquier clúster Minikube existente
minikube delete --all 2>/dev/null

# Iniciar clúster de 3 nodos
minikube start \
  --nodes=3 \
  --kubernetes-version=v1.30.0 \
  --driver=docker \
  --memory=2048 \
  --cpus=2

# Verificar que los 3 nodos están Ready
kubectl get nodes
```

**Salida esperada:**

```
NAME           STATUS   ROLES           AGE   VERSION
minikube       Ready    control-plane   60s   v1.30.0
minikube-m02   Ready    <none>          45s   v1.30.0
minikube-m03   Ready    <none>          30s   v1.30.0
```

---

## Paso 1: Desplegar la aplicación base y verificar distribución entre nodos

### Objetivo

Desplegar el Deployment `kcna-webapp` con 3 réplicas y observar cómo el scheduler distribuye los Pods entre los nodos disponibles.

### Instrucciones

1. Crea el namespace `kcna-app`:

```bash
kubectl create namespace kcna-app
```

2. Crea el manifiesto del Deployment base en `~/kcna-labs/lab04/01-webapp-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kcna-webapp
  namespace: kcna-app
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
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

> **Nota:** Reemplaza `YOUR_DOCKERHUB_USER` con tu usuario real de Docker Hub.

3. Aplica el manifiesto:

```bash
cd ~/kcna-labs/lab04
kubectl apply -f 01-webapp-deployment.yaml
```

4. Espera a que los Pods estén en estado `Running` y verifica su distribución:

```bash
kubectl get pods -n kcna-app -o wide
```

### Salida esperada

```
NAME                           READY   STATUS    RESTARTS   AGE   IP           NODE
kcna-webapp-6f8b9c7d4a-abc12  1/1     Running   0          15s   10.244.1.2   minikube-m02
kcna-webapp-6f8b9c7d4a-def34  1/1     Running   0          15s   10.244.2.3   minikube-m03
kcna-webapp-6f8b9c7d4a-ghi56  1/1     Running   0          15s   10.244.0.4   minikube
```

### Verificación

```bash
# Confirmar que los Pods están distribuidos en al menos 2 nodos diferentes
kubectl get pods -n kcna-app -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c
```

Deberías ver los Pods repartidos entre los nodos. El scheduler asignó cada Pod al nodo con mayor puntuación en la fase de scoring, distribuyendo la carga.

---

## Paso 2: Provocar Pods en estado Pending por recursos insuficientes

### Objetivo

Crear un Deployment con requests de CPU y memoria excesivos para que el scheduler no encuentre ningún nodo viable, dejando los Pods en estado `Pending`. Luego diagnosticar la causa con `kubectl describe`.

### Instrucciones

1. Crea el manifiesto `~/kcna-labs/lab04/02-overrequest-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overrequest-app
  namespace: kcna-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: overrequest-app
  template:
    metadata:
      labels:
        app: overrequest-app
    spec:
      containers:
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "2000m"
            memory: "4Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
```

2. Aplica el manifiesto:

```bash
kubectl apply -f 02-overrequest-deployment.yaml
```

3. Verifica el estado del Pod:

```bash
kubectl get pods -n kcna-app -l app=overrequest-app
```

4. Diagnostica la causa del estado `Pending`:

```bash
kubectl describe pod -n kcna-app -l app=overrequest-app | grep -A10 "Events:"
```

### Salida esperada

Estado del Pod:

```
NAME                               READY   STATUS    RESTARTS   AGE
overrequest-app-7b4f9d8c5a-x9k2m  0/1     Pending   0          20s
```

Eventos del scheduler:

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  15s   default-scheduler  0/3 nodes are available:
           1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
           2 Insufficient cpu, 2 Insufficient memory.
```

### Verificación

```bash
# Confirmar que el Pod sigue en Pending
kubectl get pods -n kcna-app -l app=overrequest-app -o jsonpath='{.items[0].status.phase}'
echo ""
```

La salida debe ser `Pending`. Esto demuestra la fase de **filtering** del scheduler: ningún nodo pasó los filtros de recursos disponibles, por lo que el Pod no puede ser asignado.

5. Elimina el Deployment problemático para liberar la cola del scheduler:

```bash
kubectl delete -f 02-overrequest-deployment.yaml
```

---

## Paso 3: Dirigir Pods a un nodo específico con nodeSelector

### Objetivo

Etiquetar el nodo `minikube-m02` con una label personalizada y usar `nodeSelector` en el Deployment para forzar que todos los Pods se ejecuten exclusivamente en ese nodo.

### Instrucciones

1. Añade una label al nodo `minikube-m02`:

```bash
kubectl label node minikube-m02 node-role=webapp
```

2. Verifica que la label se aplicó correctamente:

```bash
kubectl get node minikube-m02 --show-labels | grep node-role
```

3. Crea el manifiesto `~/kcna-labs/lab04/03-nodeselector-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-nodeselector
  namespace: kcna-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-nodeselector
  template:
    metadata:
      labels:
        app: webapp-nodeselector
    spec:
      nodeSelector:
        node-role: webapp
      containers:
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

4. Aplica el manifiesto:

```bash
kubectl apply -f 03-nodeselector-deployment.yaml
```

5. Verifica que ambos Pods se ejecutan en `minikube-m02`:

```bash
kubectl get pods -n kcna-app -l app=webapp-nodeselector -o wide
```

### Salida esperada

```
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE
webapp-nodeselector-5c8f7b9d6a-ab1cd   1/1     Running   0          10s   10.244.1.5   minikube-m02
webapp-nodeselector-5c8f7b9d6a-ef2gh   1/1     Running   0          10s   10.244.1.6   minikube-m02
```

### Verificación

```bash
# Todos los Pods deben estar en minikube-m02
NODES=$(kubectl get pods -n kcna-app -l app=webapp-nodeselector -o jsonpath='{.items[*].spec.nodeName}')
echo "Nodos asignados: $NODES"

# Verificar que solo aparece minikube-m02
echo "$NODES" | tr ' ' '\n' | sort -u
```

La salida debe mostrar únicamente `minikube-m02`. El `nodeSelector` actúa como un filtro estricto en la fase de filtering: solo los nodos con la label `node-role=webapp` son candidatos.

---

## Paso 4: Aplicar un taint a un nodo y observar el efecto en el scheduling

### Objetivo

Aplicar un taint al nodo `minikube-m03` para evitar que se programen Pods en él, y verificar que los nuevos Pods se distribuyen solo entre los nodos restantes.

### Instrucciones

1. Aplica el taint al nodo `minikube-m03`:

```bash
kubectl taint nodes minikube-m03 dedicated=infra:NoSchedule
```

2. Verifica que el taint se aplicó:

```bash
kubectl describe node minikube-m03 | grep -A3 "Taints:"
```

**Salida esperada:**

```
Taints:             dedicated=infra:NoSchedule
```

3. Crea un Deployment nuevo que no tenga toleration. Crea `~/kcna-labs/lab04/04-no-toleration-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-no-toleration
  namespace: kcna-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-no-toleration
  template:
    metadata:
      labels:
        app: webapp-no-toleration
    spec:
      containers:
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

4. Aplica el manifiesto:

```bash
kubectl apply -f 04-no-toleration-deployment.yaml
```

5. Verifica la distribución de Pods:

```bash
kubectl get pods -n kcna-app -l app=webapp-no-toleration -o wide
```

### Salida esperada

```
NAME                                    READY   STATUS    RESTARTS   AGE   IP           NODE
webapp-no-toleration-6d9f8b7c5a-12abc   1/1     Running   0          8s    10.244.0.7   minikube
webapp-no-toleration-6d9f8b7c5a-34def   1/1     Running   0          8s    10.244.1.8   minikube-m02
webapp-no-toleration-6d9f8b7c5a-56ghi   1/1     Running   0          8s    10.244.0.9   minikube
```

### Verificación

```bash
# Confirmar que ningún Pod está en minikube-m03
kubectl get pods -n kcna-app -l app=webapp-no-toleration -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | grep -c "minikube-m03"
```

La salida debe ser `0`. El taint `dedicated=infra:NoSchedule` en `minikube-m03` hace que el scheduler lo descarte en la fase de filtering para cualquier Pod que no tenga la toleration correspondiente.

---

## Paso 5: Añadir una toleration para permitir scheduling en el nodo con taint

### Objetivo

Crear un Deployment con la `toleration` correspondiente al taint de `minikube-m03` y verificar que el scheduler puede asignar Pods a ese nodo.

### Instrucciones

1. Crea el manifiesto `~/kcna-labs/lab04/05-toleration-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-with-toleration
  namespace: kcna-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-with-toleration
  template:
    metadata:
      labels:
        app: webapp-with-toleration
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "infra"
        effect: "NoSchedule"
      containers:
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

2. Aplica el manifiesto:

```bash
kubectl apply -f 05-toleration-deployment.yaml
```

3. Verifica que los Pods se distribuyen incluyendo `minikube-m03`:

```bash
kubectl get pods -n kcna-app -l app=webapp-with-toleration -o wide
```

### Salida esperada

Los Pods se distribuyen entre los 3 nodos (incluyendo `minikube-m03`):

```
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE
webapp-with-toleration-7f9a8b6c4d-11aaa   1/1     Running   0          10s   10.244.1.10   minikube-m02
webapp-with-toleration-7f9a8b6c4d-22bbb   1/1     Running   0          10s   10.244.2.11   minikube-m03
webapp-with-toleration-7f9a8b6c4d-33ccc   1/1     Running   0          10s   10.244.0.12   minikube
```

### Verificación

```bash
# Verificar que al menos un Pod está en minikube-m03
kubectl get pods -n kcna-app -l app=webapp-with-toleration -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | sort | uniq -c
```

Deberías ver `minikube-m03` en la lista. La toleration "neutraliza" el taint, permitiendo que el nodo pase la fase de filtering. Nota que la toleration no **fuerza** al Pod a ir a ese nodo; solo lo hace elegible.

---

## Paso 6: Configurar Node Affinity con preferencia suave

### Objetivo

Usar `preferredDuringSchedulingIgnoredDuringExecution` para expresar una preferencia de que los Pods se ejecuten en `minikube-m02`, sin impedir que se programen en otros nodos si `minikube-m02` no tiene capacidad.

### Instrucciones

1. Asegúrate de que la label `node-role=webapp` sigue en `minikube-m02`:

```bash
kubectl get node minikube-m02 --show-labels | grep "node-role=webapp"
```

2. Crea el manifiesto `~/kcna-labs/lab04/06-affinity-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-affinity
  namespace: kcna-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: webapp-affinity
  template:
    metadata:
      labels:
        app: webapp-affinity
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "infra"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: node-role
                operator: In
                values:
                - webapp
      containers:
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

3. Aplica el manifiesto:

```bash
kubectl apply -f 06-affinity-deployment.yaml
```

4. Verifica la distribución de Pods:

```bash
kubectl get pods -n kcna-app -l app=webapp-affinity -o wide
```

### Salida esperada

La mayoría de los Pods deberían estar en `minikube-m02` (el nodo preferido), pero algunos pueden estar en otros nodos:

```
NAME                               READY   STATUS    RESTARTS   AGE   IP            NODE
webapp-affinity-5d7f8a9b3c-aaaa1   1/1     Running   0          8s    10.244.1.13   minikube-m02
webapp-affinity-5d7f8a9b3c-bbbb2   1/1     Running   0          8s    10.244.1.14   minikube-m02
webapp-affinity-5d7f8a9b3c-cccc3   1/1     Running   0          8s    10.244.1.15   minikube-m02
webapp-affinity-5d7f8a9b3c-dddd4   1/1     Running   0          8s    10.244.2.16   minikube-m03
```

### Verificación

```bash
# Contar Pods por nodo
kubectl get pods -n kcna-app -l app=webapp-affinity -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | sort | uniq -c
```

Deberías ver una concentración mayor en `minikube-m02`. La afinidad `preferred` con peso 80 incrementa significativamente la puntuación de `minikube-m02` en la fase de scoring, pero no descarta otros nodos en la fase de filtering. Esto es la diferencia clave entre `preferred` (suave) y `required` (estricta).

---

## Paso 7 (Bonus): Verificar Node Affinity estricta con nodo inexistente

### Objetivo

Demostrar que `requiredDuringSchedulingIgnoredDuringExecution` con una label inexistente deja el Pod en `Pending`, similar al comportamiento de `nodeSelector`.

### Instrucciones

1. Crea el manifiesto `~/kcna-labs/lab04/07-required-affinity.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-required-affinity
  namespace: kcna-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-required-affinity
  template:
    metadata:
      labels:
        app: webapp-required-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role
                operator: In
                values:
                - gpu-workload
      containers:
      - name: webapp
        image: docker.io/YOUR_DOCKERHUB_USER/kcna-webapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

2. Aplica el manifiesto:

```bash
kubectl apply -f 07-required-affinity.yaml
```

3. Verifica que el Pod está en `Pending`:

```bash
kubectl get pods -n kcna-app -l app=webapp-required-affinity
```

4. Diagnostica la causa:

```bash
kubectl describe pod -n kcna-app -l app=webapp-required-affinity | grep -A5 "Events:"
```

### Salida esperada

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5s    default-scheduler  0/3 nodes are available:
           1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
           2 node(s) didn't match Pod's node affinity/selector.
```

### Verificación

El Pod permanece en `Pending` porque ningún nodo tiene la label `node-role=gpu-workload`. Esto confirma que `requiredDuringScheduling` actúa como filtro estricto.

```bash
kubectl get pods -n kcna-app -l app=webapp-required-affinity -o jsonpath='{.items[0].status.phase}'
echo ""
# Debe imprimir: Pending
```

---

## Validación y pruebas finales

Ejecuta el siguiente script de validación para confirmar que todos los escenarios del laboratorio funcionan correctamente:

```bash
#!/bin/bash
echo "=== Validación del Lab 04 ==="
echo ""

# Test 1: Clúster de 3 nodos
NODES=$(kubectl get nodes --no-headers | wc -l)
echo "[Test 1] Nodos en el clúster: $NODES (esperado: 3)"
[ "$NODES" -eq 3 ] && echo "  ✅ PASS" || echo "  ❌ FAIL"

# Test 2: Deployment base distribuido
RUNNING=$(kubectl get pods -n kcna-app -l app=kcna-webapp --field-selector=status.phase=Running --no-headers | wc -l)
echo "[Test 2] Pods kcna-webapp Running: $RUNNING (esperado: 3)"
[ "$RUNNING" -eq 3 ] && echo "  ✅ PASS" || echo "  ❌ FAIL"

# Test 3: nodeSelector funciona
NS_NODES=$(kubectl get pods -n kcna-app -l app=webapp-nodeselector -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | sort -u)
echo "[Test 3] Pods con nodeSelector en nodo(s): $NS_NODES (esperado: minikube-m02)"
[ "$NS_NODES" = "minikube-m02" ] && echo "  ✅ PASS" || echo "  ❌ FAIL"

# Test 4: Taint bloquea scheduling
TAINT_M03=$(kubectl get pods -n kcna-app -l app=webapp-no-toleration -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | grep -c "minikube-m03")
echo "[Test 4] Pods sin toleration en minikube-m03: $TAINT_M03 (esperado: 0)"
[ "$TAINT_M03" -eq 0 ] && echo "  ✅ PASS" || echo "  ❌ FAIL"

# Test 5: Toleration permite scheduling
TOL_M03=$(kubectl get pods -n kcna-app -l app=webapp-with-toleration -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | grep -c "minikube-m03")
echo "[Test 5] Pods con toleration en minikube-m03: $TOL_M03 (esperado: >=1)"
[ "$TOL_M03" -ge 1 ] && echo "  ✅ PASS" || echo "  ❌ FAIL"

# Test 6: Required affinity deja Pod Pending
REQ_PHASE=$(kubectl get pods -n kcna-app -l app=webapp-required-affinity -o jsonpath='{.items[0].status.phase}')
echo "[Test 6] Pod con required affinity inexistente: $REQ_PHASE (esperado: Pending)"
[ "$REQ_PHASE" = "Pending" ] && echo "  ✅ PASS" || echo "  ❌ FAIL"

echo ""
echo "=== Validación completada ==="
```

Guarda el script como `~/kcna-labs/lab04/validate.sh` y ejecútalo:

```bash
chmod +x ~/kcna-labs/lab04/validate.sh
~/kcna-labs/lab04/validate.sh
```

---

## Solución de problemas

### Problema 1: Los nodos worker no pasan a estado Ready

**Síntomas:** Después de ejecutar `minikube start --nodes=3`, los nodos `minikube-m02` y/o `minikube-m03` permanecen en estado `NotReady` durante más de 2 minutos.

**Causa:** Recursos insuficientes en el host. Cada nodo Minikube requiere 2 GB de RAM y 2 CPUs. Con 3 nodos, se necesitan al menos 6 GB de RAM disponibles. Docker también consume recursos adicionales.

**Solución:**

```bash
# Verificar estado detallado de los nodos
kubectl describe node minikube-m02 | grep -A5 "Conditions:"

# Si el problema persiste, eliminar y recrear con menos memoria
minikube delete --all
minikube start --nodes=3 --kubernetes-version=v1.30.0 --driver=docker --memory=1800 --cpus=2

# Esperar 60 segundos y verificar
sleep 60
kubectl get nodes
```

### Problema 2: El Pod con nodeSelector queda en Pending

**Síntomas:** El Deployment `webapp-nodeselector` tiene Pods en estado `Pending` con el evento `FailedScheduling: 0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector`.

**Causa:** La label `node-role=webapp` no se aplicó correctamente al nodo `minikube-m02`, o se escribió con un error tipográfico (por ejemplo, `node-role=Webapp` vs `node-role=webapp`).

**Solución:**

```bash
# Verificar labels actuales del nodo
kubectl get node minikube-m02 --show-labels

# Si la label no existe o tiene error, corregir
kubectl label node minikube-m02 node-role=webapp --overwrite

# Verificar que el Pod se programa automáticamente
kubectl get pods -n kcna-app -l app=webapp-nodeselector -w
```

El scheduler reintenta automáticamente en cada ciclo, por lo que una vez corregida la label, el Pod pasará de `Pending` a `Running` sin necesidad de recrearlo.

---

## Limpieza

```bash
# Eliminar todos los recursos del namespace
kubectl delete namespace kcna-app

# Remover la label del nodo
kubectl label node minikube-m02 node-role-

# Remover el taint del nodo
kubectl taint nodes minikube-m03 dedicated=infra:NoSchedule-

# Verificar limpieza
kubectl get all -n kcna-app 2>&1
kubectl describe node minikube-m03 | grep "Taints:"
```

> **Nota:** Los archivos YAML en `~/kcna-labs/lab04/` se conservan intencionalmente como referencia para el lab integrador (lab 05).

Si deseas eliminar el clúster multi-nodo completamente:

```bash
minikube delete --all
```

---

## Resumen

En este laboratorio has aplicado los conceptos fundamentales del scheduling en Kubernetes:

| Concepto | Qué aprendiste |
|----------|----------------|
| **Recursos (requests/limits)** | El scheduler usa los `requests` para decidir si un nodo tiene capacidad. Requests excesivos provocan `Pending`. |
| **nodeSelector** | Filtro estricto basado en labels de nodo. Solo nodos con la label exacta son candidatos. |
| **Taints** | Mecanismo de exclusión en el nodo. Los Pods sin toleration son rechazados en la fase de filtering. |
| **Tolerations** | Declaración en el Pod que neutraliza un taint específico, haciendo al nodo elegible. |
| **Node Affinity (preferred)** | Incrementa la puntuación de nodos preferidos en la fase de scoring sin descartar otros. |
| **Node Affinity (required)** | Actúa como filtro estricto equivalente a nodeSelector pero con expresiones más flexibles. |

### Archivos generados

```
~/kcna-labs/lab04/
├── 01-webapp-deployment.yaml
├── 02-overrequest-deployment.yaml
├── 03-nodeselector-deployment.yaml
├── 04-no-toleration-deployment.yaml
├── 05-toleration-deployment.yaml
├── 06-affinity-deployment.yaml
├── 07-required-affinity.yaml
└── validate.sh
```

### Recursos adicionales

- [Kubernetes Docs: Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes Docs: Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes Docs: Resource Management for Pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Docs: Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
