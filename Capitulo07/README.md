# 11 Práctica 7. RBAC y permisos básicos

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Directorio de trabajo** | `~/kcna-labs/lab07/` |

## Descripción General

En este laboratorio aplicarás los principios de seguridad cloud native —mínimo privilegio, defensa en profundidad y modelo de las 4C— configurando RBAC, Secrets y SecurityContexts en un clúster Kubernetes. Crearás ServiceAccounts con permisos acotados, definirás ClusterRoles de solo lectura, generarás Secrets de tipo Opaque y TLS, y verificarás que los controles de acceso funcionan correctamente usando `kubectl auth can-i`.

## Objetivos de Aprendizaje

- [ ] Crear ServiceAccounts con permisos acotados usando Roles y RoleBindings en un namespace específico
- [ ] Definir un ClusterRole de solo lectura y vincularlo a un usuario de prueba mediante ClusterRoleBinding
- [ ] Crear y consumir Secrets de tipo Opaque y kubernetes.io/tls desde un Pod como variables de entorno y volúmenes
- [ ] Aplicar un SecurityContext a nivel de Pod y contenedor para ejecutar procesos como usuario no-root
- [ ] Verificar el principio de mínimo privilegio probando acciones denegadas con `kubectl auth can-i`

## Prerrequisitos

### Conocimientos previos

- Comprensión del modelo de las 4C y principio de mínimo privilegio (Lección 7.1)
- Familiaridad con manifiestos YAML declarativos de Kubernetes
- Conocimiento básico de codificación base64 en línea de comandos
- Conceptos de autenticación y autorización en Kubernetes

### Acceso requerido

- Clúster Minikube en ejecución con RBAC habilitado (por defecto)
- `kubectl` configurado con acceso de administrador al clúster
- `openssl` disponible en el host para generación de certificados TLS
- Lab 06 completado (namespace `lab06` existente)

## Entorno del Laboratorio

### Software requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.32.0+ | Clúster Kubernetes local |
| kubectl | 1.29.2+ | Cliente de línea de comandos |
| openssl | 3.0.2+ | Generación de certificados y codificación |
| Docker Engine | 26.1.4+ | Runtime de contenedores |

### Preparación del entorno

```bash
# Crear directorio de trabajo
mkdir -p ~/kcna-labs/lab07
cd ~/kcna-labs/lab07

# Verificar que el clúster está activo
minikube status

# Verificar que RBAC está habilitado (debe retornar recursos RBAC)
kubectl api-resources | grep rbac

# Verificar acceso de administrador
kubectl auth can-i '*' '*' --all-namespaces
```

**Salida esperada** del último comando:

```
yes
```

## Paso a Paso

### Paso 1: Crear el namespace dedicado a seguridad

**Objetivo:** Establecer el namespace `lab07` como entorno aislado para los ejercicios de RBAC y seguridad.

**Instrucciones:**

1. Crea el manifiesto del namespace:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab07
  labels:
    purpose: security-lab
    chapter: "07"
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab07/namespace.yaml
```

3. Verifica la creación:

```bash
kubectl get namespace lab07 --show-labels
```

**Salida esperada:**

```
NAME    STATUS   AGE   LABELS
lab07   Active   5s    chapter=07,kubernetes.io/metadata.name=lab07,purpose=security-lab
```

**Verificación:**

```bash
kubectl get namespace lab07 -o jsonpath='{.status.phase}'
# Debe retornar: Active
```

---

### Paso 2: Crear una ServiceAccount y un Role con permisos acotados

**Objetivo:** Implementar el principio de mínimo privilegio creando una ServiceAccount `app-reader-sa` que solo pueda leer Pods en el namespace `lab07`.

**Instrucciones:**

1. Crea el manifiesto de la ServiceAccount:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader-sa
  namespace: lab07
  labels:
    role: reader
EOF
```

2. Crea el manifiesto del Role con permisos mínimos (solo get, list, watch sobre Pods):

```bash
cat <<'EOF' > ~/kcna-labs/lab07/role-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: lab07
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

3. Crea el RoleBinding que vincula el Role a la ServiceAccount:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/rolebinding-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: lab07
subjects:
- kind: ServiceAccount
  name: app-reader-sa
  namespace: lab07
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

4. Aplica todos los manifiestos:

```bash
kubectl apply -f ~/kcna-labs/lab07/serviceaccount.yaml
kubectl apply -f ~/kcna-labs/lab07/role-pod-reader.yaml
kubectl apply -f ~/kcna-labs/lab07/rolebinding-pod-reader.yaml
```

**Salida esperada:**

```
serviceaccount/app-reader-sa created
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/read-pods-binding created
```

**Verificación:**

```bash
# Verificar que la ServiceAccount puede listar pods en lab07
kubectl auth can-i list pods --namespace=lab07 --as=system:serviceaccount:lab07:app-reader-sa

# Verificar que NO puede crear pods (mínimo privilegio)
kubectl auth can-i create pods --namespace=lab07 --as=system:serviceaccount:lab07:app-reader-sa

# Verificar que NO puede listar pods en otros namespaces
kubectl auth can-i list pods --namespace=default --as=system:serviceaccount:lab07:app-reader-sa
```

**Salida esperada:**

```
yes
no
no
```

---

### Paso 3: Crear un ClusterRole de solo lectura y vincularlo a un usuario ficticio

**Objetivo:** Definir un ClusterRole `cluster-readonly` con permisos de lectura sobre nodes, namespaces y pods a nivel de clúster, y vincularlo al usuario `dev-user` mediante un ClusterRoleBinding.

**Instrucciones:**

1. Crea el manifiesto del ClusterRole:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/clusterrole-readonly.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-readonly
  labels:
    purpose: read-only-access
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces", "pods"]
  verbs: ["get", "list", "watch"]
EOF
```

2. Crea el ClusterRoleBinding para el usuario `dev-user`:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/clusterrolebinding-devuser.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-user-readonly-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-readonly
  apiGroup: rbac.authorization.k8s.io
EOF
```

3. Aplica ambos manifiestos:

```bash
kubectl apply -f ~/kcna-labs/lab07/clusterrole-readonly.yaml
kubectl apply -f ~/kcna-labs/lab07/clusterrolebinding-devuser.yaml
```

**Salida esperada:**

```
clusterrole.rbac.authorization.k8s.io/cluster-readonly created
clusterrolebinding.rbac.authorization.k8s.io/dev-user-readonly-binding created
```

4. Verifica los permisos del usuario `dev-user`:

```bash
# Puede listar nodos (permiso de clúster)
kubectl auth can-i list nodes --as=dev-user

# Puede listar pods en cualquier namespace
kubectl auth can-i list pods --all-namespaces --as=dev-user

# NO puede eliminar pods (solo lectura)
kubectl auth can-i delete pods --as=dev-user

# NO puede crear deployments
kubectl auth can-i create deployments --as=dev-user
```

**Salida esperada:**

```
yes
yes
no
no
```

**Verificación:**

```bash
# Describir el ClusterRole para confirmar las reglas
kubectl describe clusterrole cluster-readonly
```

La salida debe mostrar las tres resources (nodes, namespaces, pods) con verbos get, list, watch únicamente.

---

### Paso 4: Crear un Secret de tipo Opaque con credenciales

**Objetivo:** Generar un Secret de tipo Opaque `app-credentials` con usuario y contraseña codificados en base64 usando `openssl`.

**Instrucciones:**

1. Genera valores codificados en base64:

```bash
# Generar una contraseña aleatoria segura
export APP_PASSWORD=$(openssl rand -base64 16)
echo "Contraseña generada: $APP_PASSWORD"

# Codificar usuario y contraseña
export B64_USER=$(echo -n "app-admin" | base64)
export B64_PASS=$(echo -n "$APP_PASSWORD" | base64)

echo "Usuario (base64): $B64_USER"
echo "Contraseña (base64): $B64_PASS"
```

2. Crea el manifiesto del Secret Opaque:

```bash
cat > ~/kcna-labs/lab07/secret-opaque.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  namespace: lab07
  labels:
    app: secret-consumer
type: Opaque
data:
  username: $(echo -n "app-admin" | base64)
  password: $(echo -n "$APP_PASSWORD" | base64)
EOF
```

3. Aplica el Secret:

```bash
kubectl apply -f ~/kcna-labs/lab07/secret-opaque.yaml
```

**Salida esperada:**

```
secret/app-credentials created
```

4. Verifica el Secret (sin exponer los valores en texto claro):

```bash
kubectl get secret app-credentials -n lab07
kubectl describe secret app-credentials -n lab07
```

**Salida esperada:**

```
NAME              TYPE     DATA   AGE
app-credentials   Opaque   2      5s
```

El `describe` mostrará los campos `username` y `password` con su tamaño en bytes, sin revelar el contenido.

**Verificación:**

```bash
# Decodificar para confirmar el valor del usuario
kubectl get secret app-credentials -n lab07 -o jsonpath='{.data.username}' | base64 -d
# Debe retornar: app-admin
```

---

### Paso 5: Crear un Secret TLS con certificado autofirmado

**Objetivo:** Generar un certificado TLS autofirmado con `openssl` y almacenarlo como Secret de tipo `kubernetes.io/tls`.

**Instrucciones:**

1. Genera la clave privada y el certificado autofirmado:

```bash
cd ~/kcna-labs/lab07

# Generar clave privada RSA de 2048 bits
openssl genrsa -out tls.key 2048

# Generar certificado autofirmado válido por 365 días
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=app.lab07.local/O=KCNA-Lab"
```

2. Verifica los archivos generados:

```bash
ls -la tls.key tls.crt
openssl x509 -in tls.crt -noout -subject -dates
```

**Salida esperada (ejemplo):**

```
subject=CN = app.lab07.local, O = KCNA-Lab
notBefore=...
notAfter=...
```

3. Crea el Secret TLS usando kubectl:

```bash
kubectl create secret tls app-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=lab07
```

**Salida esperada:**

```
secret/app-tls-secret created
```

4. Verifica el Secret TLS:

```bash
kubectl get secret app-tls-secret -n lab07
kubectl describe secret app-tls-secret -n lab07
```

**Salida esperada:**

```
NAME             TYPE                DATA   AGE
app-tls-secret   kubernetes.io/tls   2      5s
```

**Verificación:**

```bash
# Confirmar que contiene las claves tls.crt y tls.key
kubectl get secret app-tls-secret -n lab07 -o jsonpath='{.data}' | jq 'keys'
# Debe retornar: ["tls.crt", "tls.key"]
```

---

### Paso 6: Crear un Pod que consuma ambos Secrets

**Objetivo:** Desplegar un Pod `secret-consumer-pod` que monte el Secret Opaque como variables de entorno y el Secret TLS como volumen, demostrando ambos métodos de consumo.

**Instrucciones:**

1. Crea el manifiesto del Pod consumidor:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/pod-secret-consumer.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-consumer-pod
  namespace: lab07
  labels:
    app: secret-consumer
spec:
  serviceAccountName: app-reader-sa
  containers:
  - name: consumer
    image: alpine:3.19.1
    command: ["sleep", "3600"]
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: app-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-credentials
          key: password
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/tls
      readOnly: true
  volumes:
  - name: tls-certs
    secret:
      secretName: app-tls-secret
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab07/pod-secret-consumer.yaml
```

3. Espera a que el Pod esté en estado Running:

```bash
kubectl wait --for=condition=Ready pod/secret-consumer-pod -n lab07 --timeout=120s
```

**Salida esperada:**

```
pod/secret-consumer-pod condition met
```

4. Verifica que las variables de entorno del Secret Opaque están disponibles:

```bash
kubectl exec secret-consumer-pod -n lab07 -- env | grep DB_
```

**Salida esperada:**

```
DB_USERNAME=app-admin
DB_PASSWORD=<valor-generado-aleatoriamente>
```

5. Verifica que el certificado TLS está montado como volumen:

```bash
kubectl exec secret-consumer-pod -n lab07 -- ls -la /etc/tls/
```

**Salida esperada:**

```
total 0
drwxrwxrwt    3 root     root           120 ...  .
drwxr-xr-x    1 root     root          4096 ...  ..
lrwxrwxrwx    1 root     root            14 ...  tls.crt -> ..data/tls.crt
lrwxrwxrwx    1 root     root            14 ...  tls.key -> ..data/tls.key
...
```

6. Verifica el contenido del certificado montado:

```bash
kubectl exec secret-consumer-pod -n lab07 -- cat /etc/tls/tls.crt | openssl x509 -noout -subject
```

**Salida esperada:**

```
subject=CN = app.lab07.local, O = KCNA-Lab
```

**Verificación:**

```bash
# Confirmar que el Pod usa la ServiceAccount correcta
kubectl get pod secret-consumer-pod -n lab07 -o jsonpath='{.spec.serviceAccountName}'
# Debe retornar: app-reader-sa
```

---

### Paso 7: Aplicar SecurityContext para ejecución no-root

**Objetivo:** Crear un Pod `secure-pod` con SecurityContext que fuerce la ejecución como usuario no-root (UID 1000), grupo 3000, filesystem de solo lectura y sin escalación de privilegios.

**Instrucciones:**

1. Crea el manifiesto del Pod seguro:

```bash
cat <<'EOF' > ~/kcna-labs/lab07/pod-secure.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: lab07
  labels:
    app: secure-app
    security: hardened
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
  - name: secure-container
    image: alpine:3.19.1
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
  volumes:
  - name: tmp-volume
    emptyDir: {}
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab07/pod-secure.yaml
```

3. Espera a que el Pod esté listo:

```bash
kubectl wait --for=condition=Ready pod/secure-pod -n lab07 --timeout=120s
```

**Salida esperada:**

```
pod/secure-pod condition met
```

4. Verifica el usuario y grupo de ejecución:

```bash
kubectl exec secure-pod -n lab07 -- id
```

**Salida esperada:**

```
uid=1000 gid=3000 groups=2000
```

5. Verifica que el filesystem raíz es de solo lectura:

```bash
kubectl exec secure-pod -n lab07 -- touch /test-file 2>&1
```

**Salida esperada:**

```
touch: /test-file: Read-only file system
```

6. Verifica que se puede escribir en el volumen temporal:

```bash
kubectl exec secure-pod -n lab07 -- touch /tmp/test-file
kubectl exec secure-pod -n lab07 -- ls -la /tmp/test-file
```

**Salida esperada:**

```
-rw-r--r--    1 1000     2000             0 ...  /tmp/test-file
```

7. Verifica que no se puede escalar privilegios:

```bash
kubectl get pod secure-pod -n lab07 -o jsonpath='{.spec.containers[0].securityContext.allowPrivilegeEscalation}'
# Debe retornar: false
```

**Verificación:**

```bash
# Confirmar todas las restricciones de seguridad
kubectl get pod secure-pod -n lab07 -o jsonpath='{.spec.securityContext}' | jq .
```

**Salida esperada:**

```json
{
  "fsGroup": 2000,
  "runAsGroup": 3000,
  "runAsNonRoot": true,
  "runAsUser": 1000
}
```

---

### Paso 8: Verificación integral del principio de mínimo privilegio

**Objetivo:** Realizar una verificación completa de todos los controles RBAC implementados, confirmando tanto los permisos concedidos como los denegados.

**Instrucciones:**

1. Verifica los permisos de la ServiceAccount `app-reader-sa`:

```bash
echo "=== Permisos de app-reader-sa en lab07 ==="
echo -n "list pods (lab07): "
kubectl auth can-i list pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
echo -n "get pods (lab07): "
kubectl auth can-i get pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
echo -n "watch pods (lab07): "
kubectl auth can-i watch pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
echo -n "create pods (lab07): "
kubectl auth can-i create pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
echo -n "delete pods (lab07): "
kubectl auth can-i delete pods -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
echo -n "list secrets (lab07): "
kubectl auth can-i list secrets -n lab07 --as=system:serviceaccount:lab07:app-reader-sa
echo -n "list pods (default): "
kubectl auth can-i list pods -n default --as=system:serviceaccount:lab07:app-reader-sa
```

**Salida esperada:**

```
=== Permisos de app-reader-sa en lab07 ===
list pods (lab07): yes
get pods (lab07): yes
watch pods (lab07): yes
create pods (lab07): no
delete pods (lab07): no
list secrets (lab07): no
list pods (default): no
```

2. Verifica los permisos del usuario `dev-user`:

```bash
echo "=== Permisos de dev-user (cluster-wide) ==="
echo -n "list nodes: "
kubectl auth can-i list nodes --as=dev-user
echo -n "list namespaces: "
kubectl auth can-i list namespaces --as=dev-user
echo -n "list pods (all-ns): "
kubectl auth can-i list pods --all-namespaces --as=dev-user
echo -n "create pods: "
kubectl auth can-i create pods --as=dev-user
echo -n "delete nodes: "
kubectl auth can-i delete nodes --as=dev-user
echo -n "create namespaces: "
kubectl auth can-i create namespaces --as=dev-user
```

**Salida esperada:**

```
=== Permisos de dev-user (cluster-wide) ===
list nodes: yes
list namespaces: yes
list pods (all-ns): yes
create pods: no
delete nodes: no
create namespaces: no
```

3. Genera un resumen de todos los recursos RBAC creados:

```bash
echo "=== Resumen de recursos RBAC en lab07 ==="
echo "--- ServiceAccounts ---"
kubectl get serviceaccounts -n lab07 --no-headers
echo ""
echo "--- Roles ---"
kubectl get roles -n lab07 --no-headers
echo ""
echo "--- RoleBindings ---"
kubectl get rolebindings -n lab07 --no-headers
echo ""
echo "--- ClusterRoles (custom) ---"
kubectl get clusterroles -l purpose=read-only-access --no-headers
echo ""
echo "--- ClusterRoleBindings (custom) ---"
kubectl get clusterrolebindings dev-user-readonly-binding --no-headers
echo ""
echo "--- Secrets ---"
kubectl get secrets -n lab07 --no-headers
echo ""
echo "--- Pods ---"
kubectl get pods -n lab07 --no-headers
```

**Salida esperada (ejemplo):**

```
=== Resumen de recursos RBAC en lab07 ===
--- ServiceAccounts ---
app-reader-sa   0         ...
default         0         ...

--- Roles ---
pod-reader   ...

--- RoleBindings ---
read-pods-binding   Role/pod-reader   ...

--- ClusterRoles (custom) ---
cluster-readonly   ...

--- ClusterRoleBindings (custom) ---
dev-user-readonly-binding   ClusterRole/cluster-readonly   ...

--- Secrets ---
app-credentials    Opaque                2   ...
app-tls-secret     kubernetes.io/tls     2   ...

--- Pods ---
secret-consumer-pod   1/1   Running   0   ...
secure-pod            1/1   Running   0   ...
```

**Verificación:**

```bash
# Verificación final: confirmar que todos los pods están Running
kubectl get pods -n lab07 -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

**Salida esperada:**

```
secret-consumer-pod	Running
secure-pod	Running
```

---

## Limpieza del Laboratorio

Ejecuta los siguientes comandos para eliminar todos los recursos creados:

```bash
# Eliminar recursos del namespace lab07
kubectl delete namespace lab07

# Eliminar recursos a nivel de clúster
kubectl delete clusterrole cluster-readonly
kubectl delete clusterrolebinding dev-user-readonly-binding

# Eliminar archivos de certificados locales
rm -f ~/kcna-labs/lab07/tls.key ~/kcna-labs/lab07/tls.crt

# Verificar limpieza
kubectl get namespace lab07 2>&1 | grep -q "not found" && echo "Namespace eliminado correctamente"
kubectl get clusterrole cluster-readonly 2>&1 | grep -q "not found" && echo "ClusterRole eliminado correctamente"
kubectl get clusterrolebinding dev-user-readonly-binding 2>&1 | grep -q "not found" && echo "ClusterRoleBinding eliminado correctamente"
```

---

## Resumen de Conceptos Aplicados

| Concepto | Recurso Creado | Principio de Seguridad |
|----------|---------------|----------------------|
| ServiceAccount | `app-reader-sa` | Identidad mínima para workloads |
| Role + RoleBinding | `pod-reader` / `read-pods-binding` | Mínimo privilegio (namespace-scoped) |
| ClusterRole + ClusterRoleBinding | `cluster-readonly` / `dev-user-readonly-binding` | Lectura global sin escritura |
| Secret Opaque | `app-credentials` | Protección de credenciales en reposo |
| Secret TLS | `app-tls-secret` | Cifrado en tránsito (certificados) |
| SecurityContext | `secure-pod` | Defensa en profundidad (non-root, read-only fs) |
| `kubectl auth can-i` | Verificaciones | Auditoría de permisos efectivos |

## Troubleshooting

| Problema | Causa Probable | Solución |
|----------|---------------|----------|
| `kubectl auth can-i` retorna `yes` cuando debería ser `no` | Estás usando el contexto de admin sin `--as` | Asegúrate de incluir `--as=system:serviceaccount:lab07:app-reader-sa` o `--as=dev-user` |
| Pod `secret-consumer-pod` en estado `ImagePullBackOff` | Imagen `alpine:3.19.1` no disponible | Ejecuta `minikube ssh -- docker pull alpine:3.19.1` o usa `alpine:latest` |
| Pod `secure-pod` en `CrashLoopBackOff` | El comando `sleep` no existe en la imagen | Verifica que usas `alpine:3.19.1` que incluye `sleep` de busybox |
| Error `forbidden` al crear ClusterRole | El usuario actual no tiene permisos de admin | Verifica con `kubectl auth can-i '*' '*' --all-namespaces` que retorne `yes` |
| Secret TLS falla con `tls: failed to find any PEM data` | Archivos `tls.crt` o `tls.key` vacíos o corruptos | Regenera los certificados con los comandos de openssl del Paso 5 |
| Error `namespace lab07 not found` al aplicar recursos | El namespace no fue creado primero | Ejecuta primero `kubectl apply -f ~/kcna-labs/lab07/namespace.yaml` |
