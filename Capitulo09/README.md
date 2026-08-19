# 10 Práctica 9. Diagnóstico de fallas comunes

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Prerrequisitos de labs** | Lab 06, Lab 07, Lab 08 completados |

## Visión General

Este laboratorio final integra todas las habilidades de troubleshooting del curso. Se introducirán deliberadamente **cinco escenarios de fallo** distribuidos en múltiples namespaces que cubren: imágenes inexistentes (ImagePullBackOff), configuración errónea de contenedores (CrashLoopBackOff), selectores de Service incorrectos, NetworkPolicies bloqueantes, PVCs en estado Pending por StorageClass inexistente y errores de permisos RBAC. Aplicarás la metodología sistemática de cinco pasos — Observar → Localizar → Describir → Inspeccionar logs → Actuar y verificar — para diagnosticar cada fallo, documentar la causa raíz y aplicar la corrección mínima necesaria.

## Objetivos de Aprendizaje

- [ ] Diagnosticar y resolver Pods en estado CrashLoopBackOff e ImagePullBackOff utilizando `kubectl describe` y `kubectl logs`
- [ ] Identificar y corregir problemas de networking: Service con selector incorrecto y NetworkPolicy bloqueante
- [ ] Detectar y resolver un PVC en estado Pending por StorageClass incorrecta
- [ ] Diagnosticar un error de permisos RBAC que impide a una ServiceAccount acceder a recursos del clúster
- [ ] Aplicar una metodología sistemática de troubleshooting usando `kubectl describe`, `logs`, `exec` y análisis de eventos

## Prerrequisitos

### Conocimientos Requeridos

| Tema | Nivel |
|------|-------|
| Comandos kubectl (get, describe, logs, exec, apply, delete) | Intermedio |
| Objetos Kubernetes: Pods, Deployments, Services, PVCs, RBAC | Intermedio |
| NetworkPolicies | Básico |
| Metodología de troubleshooting de 5 pasos | Básico |

### Acceso y Software

| Componente | Versión / Requisito |
|------------|-------------------|
| Minikube ejecutándose | 1.32.0+ |
| kubectl configurado con acceso admin | 1.29.2+ |
| jq instalado | 1.6 |
| Labs 06, 07 y 08 completados | Namespaces `lab06`, `lab07`, `lab08` existentes |

## Entorno del Laboratorio

### Verificación Inicial

```bash
# Verificar que Minikube está corriendo
minikube status

# Verificar acceso de kubectl
kubectl cluster-info

# Verificar namespaces de labs anteriores
kubectl get ns lab06 lab07 lab08

# Verificar jq
jq --version
```

**Salida esperada** (parcial):

```
NAME    STATUS   AGE
lab06   Active   ...
lab07   Active   ...
lab08   Active   ...
```

### Crear Directorio de Trabajo

```bash
mkdir -p ~/kcna-labs/lab09
cd ~/kcna-labs/lab09
```

---

## Paso 1: Preparación — Inyectar los Escenarios de Fallo

**Objetivo:** Crear los recursos deliberadamente rotos que servirán como escenarios de diagnóstico.

### Instrucciones

1. Crear el namespace dedicado para los escenarios de fallo de Pods:

```bash
kubectl create namespace lab09-broken
```

2. Crear el archivo con todos los recursos rotos:

```bash
cat > ~/kcna-labs/lab09/broken-scenarios.yaml << 'EOF'
---
# ESCENARIO 1a: Pod con imagen inexistente → ImagePullBackOff
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  namespace: lab09-broken
  labels:
    scenario: image-pull
spec:
  containers:
  - name: nginx-bad
    image: nginx:99.99.99
    ports:
    - containerPort: 80
---
# ESCENARIO 1b: Pod con variable de entorno mal configurada → CrashLoopBackOff
apiVersion: v1
kind: Pod
metadata:
  name: config-crash-pod
  namespace: lab09-broken
  labels:
    scenario: crash-loop
spec:
  containers:
  - name: alpine-app
    image: alpine:3.19.1
    command: ["/bin/sh", "-c"]
    args:
    - |
      if [ -z "$REQUIRED_CONFIG" ]; then
        echo "ERROR: REQUIRED_CONFIG variable is not set" >&2
        exit 1
      fi
      echo "App running with config: $REQUIRED_CONFIG"
      sleep 3600
    env:
    - name: APP_NAME
      value: "broken-app"
    # FALLO: Falta la variable REQUIRED_CONFIG
---
# ESCENARIO 2: Service con selector incorrecto en lab06
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
  namespace: lab06
spec:
  selector:
    app: web-app-TYPO
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
---
# ESCENARIO 3: NetworkPolicy que bloquea todo el tráfico en lab06
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-all
  namespace: lab06
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: []
---
# ESCENARIO 4: PVC con StorageClass inexistente en lab08
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-broken
  namespace: lab08
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: nonexistent-sc
  resources:
    requests:
      storage: 1Gi
---
# ESCENARIO 5: Pod que intenta crear ConfigMap sin permisos en lab07
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: lab07
spec:
  serviceAccountName: app-reader-sa
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:1.29
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Intentando crear un ConfigMap con ServiceAccount app-reader-sa..."
      kubectl create configmap test-cm --from-literal=key=value -n lab07 2>&1
      echo "Resultado del intento registrado. Entrando en sleep..."
      sleep 3600
EOF
```

3. Aplicar todos los escenarios de fallo:

```bash
kubectl apply -f ~/kcna-labs/lab09/broken-scenarios.yaml
```

**Salida esperada:**

```
pod/crash-pod created
pod/config-crash-pod created
service/broken-svc created
networkpolicy.networking.k8s.io/block-all created
persistentvolumeclaim/pvc-broken created
pod/rbac-test-pod created
```

4. Esperar 30 segundos para que los fallos se manifiesten:

```bash
sleep 30
```

### Verificación

```bash
# Vista rápida de los Pods problemáticos
kubectl get pods -n lab09-broken
kubectl get pods -n lab07 rbac-test-pod
kubectl get pvc -n lab08 pvc-broken
```

Deberías observar estados como `ImagePullBackOff`, `CrashLoopBackOff` y `Pending`.

---

## Paso 2: Escenario 1a — Diagnosticar ImagePullBackOff

**Objetivo:** Identificar por qué el Pod `crash-pod` no puede iniciar y corregir la imagen.

### Instrucciones

1. **Observar** el síntoma:

```bash
kubectl get pod crash-pod -n lab09-broken
```

**Salida esperada:**

```
NAME        READY   STATUS             RESTARTS   AGE
crash-pod   0/1     ImagePullBackOff   0          ~1m
```

2. **Describir** el Pod para obtener detalles del evento:

```bash
kubectl describe pod crash-pod -n lab09-broken | tail -20
```

**Salida esperada** (sección Events):

```
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  1m    default-scheduler  Successfully assigned lab09-broken/crash-pod to ...
  Normal   Pulling    1m    kubelet            Pulling image "nginx:99.99.99"
  Warning  Failed     1m    kubelet            Failed to pull image "nginx:99.99.99": ... not found
  Warning  Failed     1m    kubelet            Error: ErrImagePull
  Normal   BackOff    30s   kubelet            Back-off pulling image "nginx:99.99.99"
  Warning  Failed     30s   kubelet            Error: ImagePullBackOff
```

3. **Documentar la causa raíz:**

```bash
cat > ~/kcna-labs/lab09/diagnostico-1a.md << 'EOF'
# Diagnóstico: crash-pod (ImagePullBackOff)

## Síntoma
Pod en estado ImagePullBackOff, 0 reinicios.

## Causa Raíz
La imagen `nginx:99.99.99` no existe en Docker Hub.
El tag 99.99.99 es inválido.

## Corrección
Cambiar la imagen a una versión válida: `nginx:1.25.4`
EOF
```

4. **Actuar** — corregir la imagen:

```bash
kubectl set image pod/crash-pod nginx-bad=nginx:1.25.4 -n lab09-broken
```

> **Nota:** `kubectl set image` no funciona directamente en Pods standalone. Debemos eliminar y recrear:

```bash
kubectl delete pod crash-pod -n lab09-broken

cat > ~/kcna-labs/lab09/crash-pod-fixed.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  namespace: lab09-broken
  labels:
    scenario: image-pull
spec:
  containers:
  - name: nginx-bad
    image: nginx:1.25.4
    ports:
    - containerPort: 80
EOF

kubectl apply -f ~/kcna-labs/lab09/crash-pod-fixed.yaml
```

5. **Verificar** la corrección:

```bash
kubectl wait --for=condition=Ready pod/crash-pod -n lab09-broken --timeout=60s
kubectl get pod crash-pod -n lab09-broken
```

**Salida esperada:**

```
NAME        READY   STATUS    RESTARTS   AGE
crash-pod   1/1     Running   0          15s
```

---

## Paso 3: Escenario 1b — Diagnosticar CrashLoopBackOff

**Objetivo:** Identificar por qué `config-crash-pod` reinicia continuamente y corregir la configuración.

### Instrucciones

1. **Observar** el estado:

```bash
kubectl get pod config-crash-pod -n lab09-broken
```

**Salida esperada:**

```
NAME               READY   STATUS             RESTARTS      AGE
config-crash-pod   0/1     CrashLoopBackOff   4 (20s ago)   2m
```

2. **Inspeccionar logs** del contenedor que falló (usando `--previous` si ya reinició):

```bash
kubectl logs config-crash-pod -n lab09-broken --previous
```

**Salida esperada:**

```
ERROR: REQUIRED_CONFIG variable is not set
```

3. **Describir** para confirmar el patrón de reinicio:

```bash
kubectl describe pod config-crash-pod -n lab09-broken | grep -A 5 "Last State"
```

**Salida esperada:**

```
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

4. **Documentar la causa raíz:**

```bash
cat > ~/kcna-labs/lab09/diagnostico-1b.md << 'EOF'
# Diagnóstico: config-crash-pod (CrashLoopBackOff)

## Síntoma
Pod reiniciando continuamente con exit code 1.

## Causa Raíz
El script del contenedor requiere la variable de entorno REQUIRED_CONFIG,
pero solo se definió APP_NAME. El contenedor sale con error al no encontrarla.

## Corrección
Añadir la variable de entorno REQUIRED_CONFIG al Pod.
EOF
```

5. **Actuar** — recrear el Pod con la variable correcta:

```bash
kubectl delete pod config-crash-pod -n lab09-broken

cat > ~/kcna-labs/lab09/config-crash-pod-fixed.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: config-crash-pod
  namespace: lab09-broken
  labels:
    scenario: crash-loop
spec:
  containers:
  - name: alpine-app
    image: alpine:3.19.1
    command: ["/bin/sh", "-c"]
    args:
    - |
      if [ -z "$REQUIRED_CONFIG" ]; then
        echo "ERROR: REQUIRED_CONFIG variable is not set" >&2
        exit 1
      fi
      echo "App running with config: $REQUIRED_CONFIG"
      sleep 3600
    env:
    - name: APP_NAME
      value: "broken-app"
    - name: REQUIRED_CONFIG
      value: "production-settings-v1"
EOF

kubectl apply -f ~/kcna-labs/lab09/config-crash-pod-fixed.yaml
```

6. **Verificar:**

```bash
kubectl wait --for=condition=Ready pod/config-crash-pod -n lab09-broken --timeout=60s
kubectl logs config-crash-pod -n lab09-broken
```

**Salida esperada:**

```
App running with config: production-settings-v1
```

---

## Paso 4: Escenario 2 — Diagnosticar Service con Selector Incorrecto

**Objetivo:** Identificar por qué `broken-svc` no tiene endpoints y corregir el selector.

### Instrucciones

1. **Observar** — verificar que el Service no tiene endpoints:

```bash
kubectl get endpoints broken-svc -n lab06
```

**Salida esperada:**

```
NAME         ENDPOINTS   AGE
broken-svc   <none>      3m
```

2. **Describir** el Service para ver el selector configurado:

```bash
kubectl describe svc broken-svc -n lab06 | grep -A 2 "Selector"
```

**Salida esperada:**

```
Selector:          app=web-app-TYPO
```

3. **Localizar** los labels reales de los Pods del Deployment `web-app`:

```bash
kubectl get pods -n lab06 --show-labels | grep web-app
```

**Salida esperada** (los Pods tienen label `app=web-app`, no `app=web-app-TYPO`):

```
web-app-xxxxx   1/1   Running   ...   app=web-app,...
```

4. **Documentar la causa raíz:**

```bash
cat > ~/kcna-labs/lab09/diagnostico-2.md << 'EOF'
# Diagnóstico: broken-svc (Sin Endpoints)

## Síntoma
El Service broken-svc muestra <none> en ENDPOINTS.

## Causa Raíz
El selector del Service es `app=web-app-TYPO` pero los Pods
del Deployment web-app tienen el label `app=web-app`.
El typo "TYPO" en el selector impide el match.

## Corrección
Cambiar el selector a `app=web-app`.
EOF
```

5. **Actuar** — parchear el Service:

```bash
kubectl patch svc broken-svc -n lab06 --type='json' \
  -p='[{"op": "replace", "path": "/spec/selector/app", "value": "web-app"}]'
```

6. **Verificar** que ahora tiene endpoints:

```bash
kubectl get endpoints broken-svc -n lab06
```

**Salida esperada:**

```
NAME         ENDPOINTS          AGE
broken-svc   10.244.x.x:8080   4m
```

7. **Prueba de conectividad** (opcional, desde un Pod de diagnóstico):

```bash
kubectl run test-curl --rm -i --restart=Never -n lab06 \
  --image=busybox:1.36.1 -- wget -qO- http://broken-svc.lab06.svc.cluster.local:80 2>/dev/null || echo "Conexión bloqueada (posiblemente por NetworkPolicy - se resuelve en el siguiente paso)"
```

> **Nota:** Es probable que la conexión falle por la NetworkPolicy `block-all`. Esto se corrige en el Paso 5.

---

## Paso 5: Escenario 3 — Diagnosticar NetworkPolicy Bloqueante

**Objetivo:** Identificar la NetworkPolicy que bloquea todo el tráfico entrante en `lab06` y corregirla.

### Instrucciones

1. **Observar** — intentar acceder al Service desde dentro del clúster:

```bash
kubectl run diag-pod --rm -i --restart=Never -n lab06 \
  --image=busybox:1.36.1 -- sh -c "wget -qO- --timeout=5 http://broken-svc:80 2>&1 || echo 'TIMEOUT: tráfico bloqueado'"
```

**Salida esperada:**

```
TIMEOUT: tráfico bloqueado
```

2. **Localizar** las NetworkPolicies en el namespace:

```bash
kubectl get networkpolicies -n lab06
```

**Salida esperada:**

```
NAME        POD-SELECTOR   AGE
block-all   <none>         5m
```

3. **Describir** la NetworkPolicy:

```bash
kubectl describe networkpolicy block-all -n lab06
```

**Salida esperada:**

```
Name:         block-all
Namespace:    lab06
Spec:
  PodSelector:     <none> (Coverage: all pods in the namespace)
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Not affecting egress traffic
  Policy Types: Ingress
```

4. **Documentar la causa raíz:**

```bash
cat > ~/kcna-labs/lab09/diagnostico-3.md << 'EOF'
# Diagnóstico: NetworkPolicy block-all

## Síntoma
Ningún Pod en lab06 puede recibir tráfico entrante.
wget/curl a Services internos resulta en timeout.

## Causa Raíz
La NetworkPolicy "block-all" selecciona TODOS los Pods (podSelector vacío)
y define policyTypes: [Ingress] con una lista de ingress vacía ([]).
Esto bloquea TODO el tráfico entrante a todos los Pods del namespace.

## Corrección
Eliminar la NetworkPolicy bloqueante. En producción se reemplazaría
por una política que permita tráfico desde fuentes específicas.
EOF
```

5. **Actuar** — eliminar la NetworkPolicy bloqueante:

```bash
kubectl delete networkpolicy block-all -n lab06
```

6. **Verificar** la conectividad restaurada:

```bash
kubectl run diag-pod2 --rm -i --restart=Never -n lab06 \
  --image=busybox:1.36.1 -- sh -c "wget -qO- --timeout=5 http://broken-svc:80 2>&1 | head -5"
```

**Salida esperada:** Debería mostrar la respuesta HTTP del Pod de `web-app` (HTML o JSON según la aplicación desplegada en lab06).

---

## Paso 6: Escenario 4 — Diagnosticar PVC en Estado Pending

**Objetivo:** Identificar por qué `pvc-broken` no se vincula a ningún PV y corregir la StorageClass.

### Instrucciones

1. **Observar** el estado del PVC:

```bash
kubectl get pvc pvc-broken -n lab08
```

**Salida esperada:**

```
NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS     AGE
pvc-broken   Pending                                      nonexistent-sc   6m
```

2. **Describir** el PVC para ver los eventos:

```bash
kubectl describe pvc pvc-broken -n lab08 | grep -A 10 "Events"
```

**Salida esperada:**

```
Events:
  Type     Reason              Age    From                         Message
  ----     ------              ----   ----                         -------
  Warning  ProvisioningFailed  1m     persistentvolume-controller  storageclass.storage.k8s.io "nonexistent-sc" not found
```

3. **Identificar** las StorageClasses disponibles:

```bash
kubectl get storageclass
```

**Salida esperada:**

```
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  ...
```

4. **Documentar la causa raíz:**

```bash
cat > ~/kcna-labs/lab09/diagnostico-4.md << 'EOF'
# Diagnóstico: pvc-broken (Pending)

## Síntoma
PVC pvc-broken en estado Pending indefinidamente.

## Causa Raíz
El PVC solicita storageClassName: "nonexistent-sc" que no existe
en el clúster. La única StorageClass disponible es "standard".

## Corrección
Eliminar el PVC y recrearlo con storageClassName: "standard".
EOF
```

5. **Actuar** — eliminar y recrear con la StorageClass correcta:

```bash
kubectl delete pvc pvc-broken -n lab08

cat > ~/kcna-labs/lab09/pvc-broken-fixed.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-broken
  namespace: lab08
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f ~/kcna-labs/lab09/pvc-broken-fixed.yaml
```

6. **Verificar** que el PVC se vincula:

```bash
kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/pvc-broken -n lab08 --timeout=30s
kubectl get pvc pvc-broken -n lab08
```

**Salida esperada:**

```
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-broken   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            standard       10s
```

---

## Paso 7: Escenario 5 — Diagnosticar Error de Permisos RBAC

**Objetivo:** Identificar por qué la ServiceAccount `app-reader-sa` no puede crear ConfigMaps y documentar la limitación.

### Instrucciones

1. **Observar** los logs del Pod que intenta la operación:

```bash
kubectl logs rbac-test-pod -n lab07
```

**Salida esperada:**

```
Intentando crear un ConfigMap con ServiceAccount app-reader-sa...
error: failed to create configmap: configmaps is forbidden: User "system:serviceaccount:lab07:app-reader-sa" cannot create resource "configmaps" in API group "" in the namespace "lab07"
Resultado del intento registrado. Entrando en sleep...
```

2. **Verificar** los permisos de la ServiceAccount con `kubectl auth can-i`:

```bash
# Verificar si puede crear configmaps
kubectl auth can-i create configmaps -n lab07 --as=system:serviceaccount:lab07:app-reader-sa

# Verificar qué SÍ puede hacer (listar pods)
kubectl auth can-i get pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa

kubectl auth can-i list pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
```

**Salida esperada:**

```
no
yes
yes
```

3. **Describir** el Role asociado para confirmar los permisos definidos:

```bash
kubectl describe role pod-reader -n lab07
```

**Salida esperada:**

```
Name:         pod-reader
Namespace:    lab07
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list watch]
```

4. **Describir** el RoleBinding para confirmar la asociación:

```bash
kubectl get rolebinding -n lab07 -o wide
```

5. **Documentar la causa raíz:**

```bash
cat > ~/kcna-labs/lab09/diagnostico-5.md << 'EOF'
# Diagnóstico: rbac-test-pod (Forbidden)

## Síntoma
El Pod rbac-test-pod no puede crear ConfigMaps.
Error: "configmaps is forbidden"

## Causa Raíz
La ServiceAccount "app-reader-sa" está vinculada al Role "pod-reader"
que solo permite verbos [get, list, watch] sobre el recurso "pods".
No tiene permisos para "create" sobre "configmaps".
Esto es CORRECTO según el principio de mínimo privilegio.

## Resolución
Este NO es un bug sino un control de seguridad funcionando correctamente.
Si se necesitara crear ConfigMaps, se debería crear un Role adicional
con permisos específicos y vincularlo mediante un nuevo RoleBinding.
Para este lab, creamos el Role adicional para demostrar la corrección.
EOF
```

6. **Actuar** — crear un Role adicional que permita crear ConfigMaps (demostración controlada):

```bash
cat > ~/kcna-labs/lab09/rbac-fix.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-creator
  namespace: lab07
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-configmap-binding
  namespace: lab07
subjects:
- kind: ServiceAccount
  name: app-reader-sa
  namespace: lab07
roleRef:
  kind: Role
  name: configmap-creator
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f ~/kcna-labs/lab09/rbac-fix.yaml
```

7. **Verificar** que ahora tiene permiso:

```bash
kubectl auth can-i create configmaps -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
```

**Salida esperada:**

```
yes
```

8. **Verificar** ejecutando la acción desde dentro del Pod:

```bash
kubectl exec rbac-test-pod -n lab07 -- kubectl create configmap test-cm \
  --from-literal=key=value -n lab07 2>&1 || true
```

**Salida esperada:**

```
configmap/test-cm created
```

---

## Validación y Testing Final

### Script de Validación Completa

Ejecuta el siguiente script para verificar que todos los escenarios han sido corregidos:

```bash
cat > ~/kcna-labs/lab09/validate-all.sh << 'SCRIPT'
#!/bin/bash
echo "=========================================="
echo " VALIDACIÓN FINAL - Lab 09"
echo "=========================================="
PASS=0
FAIL=0

# Test 1a: crash-pod Running
echo -n "[1a] crash-pod en Running: "
STATUS=$(kubectl get pod crash-pod -n lab09-broken -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$STATUS" = "Running" ]; then echo "✅ PASS"; ((PASS++)); else echo "❌ FAIL (estado: $STATUS)"; ((FAIL++)); fi

# Test 1b: config-crash-pod Running
echo -n "[1b] config-crash-pod en Running: "
STATUS=$(kubectl get pod config-crash-pod -n lab09-broken -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$STATUS" = "Running" ]; then echo "✅ PASS"; ((PASS++)); else echo "❌ FAIL (estado: $STATUS)"; ((FAIL++)); fi

# Test 2: broken-svc tiene endpoints
echo -n "[2]  broken-svc tiene endpoints: "
EP=$(kubectl get endpoints broken-svc -n lab06 -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)
if [ -n "$EP" ]; then echo "✅ PASS ($EP)"; ((PASS++)); else echo "❌ FAIL (sin endpoints)"; ((FAIL++)); fi

# Test 3: No existe NetworkPolicy block-all
echo -n "[3]  NetworkPolicy block-all eliminada: "
NP=$(kubectl get networkpolicy block-all -n lab06 2>&1)
if echo "$NP" | grep -q "NotFound"; then echo "✅ PASS"; ((PASS++)); else echo "❌ FAIL (aún existe)"; ((FAIL++)); fi

# Test 4: pvc-broken en Bound
echo -n "[4]  pvc-broken en Bound: "
PVC_STATUS=$(kubectl get pvc pvc-broken -n lab08 -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$PVC_STATUS" = "Bound" ]; then echo "✅ PASS"; ((PASS++)); else echo "❌ FAIL (estado: $PVC_STATUS)"; ((FAIL++)); fi

# Test 5: app-reader-sa puede crear configmaps
echo -n "[5]  RBAC: app-reader-sa puede crear configmaps: "
CAN=$(kubectl auth can-i create configmaps -n lab07 --as=system:serviceaccount:lab07:app-reader-sa 2>/dev/null)
if [ "$CAN" = "yes" ]; then echo "✅ PASS"; ((PASS++)); else echo "❌ FAIL ($CAN)"; ((FAIL++)); fi

echo ""
echo "=========================================="
echo " RESULTADO: $PASS/6 pruebas pasaron"
echo "=========================================="
if [ $FAIL -eq 0 ]; then
  echo " 🎉 ¡TODOS LOS ESCENARIOS RESUELTOS!"
else
  echo " ⚠️  Revisa los escenarios que fallaron."
fi
SCRIPT

chmod +x ~/kcna-labs/lab09/validate-all.sh
bash ~/kcna-labs/lab09/validate-all.sh
```

**Salida esperada:**

```
==========================================
 VALIDACIÓN FINAL - Lab 09
==========================================
[1a] crash-pod en Running: ✅ PASS
[1b] config-crash-pod en Running: ✅ PASS
[2]  broken-svc tiene endpoints: ✅ PASS (10.244.x.x)
[3]  NetworkPolicy block-all eliminada: ✅ PASS
[4]  pvc-broken en Bound: ✅ PASS
[5]  RBAC: app-reader-sa puede crear configmaps: ✅ PASS

==========================================
 RESULTADO: 6/6 pruebas pasaron
==========================================
 🎉 ¡TODOS LOS ESCENARIOS RESUELTOS!
```

### Verificación con Eventos del Clúster

Para una vista consolidada de todos los eventos recientes:

```bash
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
```

---

## Troubleshooting del Laboratorio

### Problema 1: El Pod `rbac-test-pod` no inicia (ImagePullBackOff en bitnami/kubectl)

**Síntoma:** El Pod `rbac-test-pod` queda en `ImagePullBackOff` o `ErrImagePull` en lugar de ejecutarse.

**Causa:** La imagen `bitnami/kubectl:1.29` no puede descargarse por restricciones de red, rate limiting de Docker Hub, o la versión exacta no está disponible.

**Solución:**

```bash
# Verificar el error exacto
kubectl describe pod rbac-test-pod -n lab07 | grep -A 3 "Events"

# Si es rate limiting, usar la imagen con SHA o tag alternativo
kubectl delete pod rbac-test-pod -n lab07

# Recrear con imagen alternativa
cat > /tmp/rbac-test-pod-alt.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: lab07
spec:
  serviceAccountName: app-reader-sa
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Intentando crear un ConfigMap con ServiceAccount app-reader-sa..."
      kubectl create configmap test-cm --from-literal=key=value -n lab07 2>&1
      echo "Resultado del intento registrado. Entrando en sleep..."
      sleep 3600
EOF
kubectl apply -f /tmp/rbac-test-pod-alt.yaml
```

### Problema 2: El PVC `pvc-broken` permanece en Pending incluso después de corregir la StorageClass

**Síntoma:** Después de recrear el PVC con `storageClassName: standard`, sigue en estado `Pending`.

**Causa:** El addon `storage-provisioner` de Minikube no está habilitado o el provisioner está en error.

**Solución:**

```bash
# Verificar que el addon de storage está habilitado
minikube addons list | grep storage

# Si no está habilitado:
minikube addons enable storage-provisioner

# Verificar que el provisioner Pod está corriendo
kubectl get pods -n kube-system | grep storage

# Si el problema persiste, crear un PV manualmente:
cat > /tmp/manual-pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv-for-broken
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  hostPath:
    path: /tmp/lab09-pv-data
EOF
kubectl apply -f /tmp/manual-pv.yaml

# Verificar binding
kubectl get pvc pvc-broken -n lab08
```

---

## Limpieza

```bash
# Eliminar namespace de escenarios rotos
kubectl delete namespace lab09-broken

# Eliminar recursos añadidos en otros namespaces
kubectl delete svc broken-svc -n lab06 --ignore-not-found
kubectl delete pvc pvc-broken -n lab08 --ignore-not-found
kubectl delete pod rbac-test-pod -n lab07 --ignore-not-found
kubectl delete configmap test-cm -n lab07 --ignore-not-found
kubectl delete role configmap-creator -n lab07 --ignore-not-found
kubectl delete rolebinding app-reader-configmap-binding -n lab07 --ignore-not-found

# Verificar limpieza
echo "--- Verificación de limpieza ---"
kubectl get all -n lab09-broken 2>&1
kubectl get pvc pvc-broken -n lab08 2>&1
kubectl get pod rbac-test-pod -n lab07 2>&1
```

**Salida esperada:**

```
namespace "lab09-broken" deleted
...
Error from server (NotFound): namespaces "lab09-broken" not found
Error from server (NotFound): ...
```

---

## Resumen

### Habilidades Practicadas

| Escenario | Estado de Fallo | Herramienta Principal | Acción Correctiva |
|-----------|----------------|----------------------|-------------------|
| 1a | ImagePullBackOff | `kubectl describe pod` (Events) | Corregir tag de imagen |
| 1b | CrashLoopBackOff | `kubectl logs --previous` | Añadir variable de entorno faltante |
| 2 | Service sin Endpoints | `kubectl get endpoints` + labels | Corregir selector del Service |
| 3 | Tráfico bloqueado | `kubectl describe networkpolicy` | Eliminar NetworkPolicy restrictiva |
| 4 | PVC Pending | `kubectl describe pvc` + `get sc` | Corregir StorageClass |
| 5 | RBAC Forbidden | `kubectl auth can-i` | Crear Role y RoleBinding adicional |

### Metodología Aplicada

En cada escenario se siguió el flujo de cinco pasos:

1. **Observar** → `kubectl get` para identificar el síntoma (estado anormal)
2. **Localizar** → Identificar el recurso y namespace afectado
3. **Describir** → `kubectl describe` para leer eventos y condiciones
4. **Inspeccionar logs** → `kubectl logs` / `kubectl auth can-i` para detalles internos
5. **Actuar y verificar** → Cambio mínimo + confirmación de resolución

### Comandos Clave Utilizados

```bash
kubectl get pods -n <ns>                          # Observar estado
kubectl describe pod <name> -n <ns>               # Eventos detallados
kubectl logs <pod> -n <ns> --previous             # Logs de contenedor crasheado
kubectl get endpoints <svc> -n <ns>               # Verificar Service→Pod binding
kubectl get networkpolicies -n <ns>               # Listar políticas de red
kubectl describe pvc <name> -n <ns>               # Estado de almacenamiento
kubectl auth can-i <verb> <resource> --as=<sa>    # Verificar permisos RBAC
kubectl get events --sort-by='.lastTimestamp'      # Eventos cronológicos
```

### Recursos Adicionales

- [Troubleshooting de Aplicaciones — Documentación oficial](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debugging Pods — Kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debugging Services — Kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [RBAC Authorization — Kubernetes.io](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies — Kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
