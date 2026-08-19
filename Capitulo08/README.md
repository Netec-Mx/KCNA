# 9 Práctica 8. Persistencia básica con PVC

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 40 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Tecnologías clave** | PersistentVolume, PersistentVolumeClaim, StorageClass, MySQL 8.3.0, emptyDir |

## Descripción General

En este laboratorio aplicarás los conceptos de almacenamiento persistente en Kubernetes creando un PersistentVolume manual con hostPath, enlazándolo a un PersistentVolumeClaim y desplegando una base de datos MySQL que sobreviva a la eliminación y recreación de su Pod. Además, contrastarás el comportamiento del almacenamiento persistente (PVC) frente al efímero (emptyDir) mediante un segundo Pod de prueba con aprovisionamiento dinámico.

## Objetivos de Aprendizaje

- [ ] Crear un PersistentVolume manual de tipo hostPath y un PersistentVolumeClaim con AccessMode ReadWriteOnce
- [ ] Desplegar un Pod MySQL 8.3.0 que utilice un PVC para persistir datos en `/var/lib/mysql`
- [ ] Verificar la persistencia de datos eliminando y recreando el Pod de base de datos
- [ ] Utilizar la StorageClass por defecto de Minikube para aprovisionamiento dinámico de un segundo PVC
- [ ] Diferenciar en práctica el comportamiento de almacenamiento efímero (emptyDir) versus persistente (PVC)

## Prerrequisitos

### Conocimientos Previos

| Concepto | Nivel |
|----------|-------|
| Ciclo de vida de Pods en Kubernetes | Intermedio |
| Manifiestos YAML declarativos | Intermedio |
| Comandos básicos de kubectl (apply, get, describe, exec, delete) | Intermedio |
| Conceptos de almacenamiento efímero vs persistente | Básico |
| Comandos SQL básicos (CREATE DATABASE, INSERT, SELECT) | Básico |

### Acceso y Entorno

- Clúster Minikube en ejecución con addon `storage-provisioner` habilitado (por defecto en Minikube 1.32.0)
- kubectl configurado con acceso de administrador al clúster
- Namespaces `lab06` y `lab07` existentes de laboratorios previos
- Conexión a Internet para descargar imágenes `mysql:8.3.0` y `busybox:1.36.1`

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Minikube | 1.32.0+ | Clúster Kubernetes local |
| kubectl | 1.29.2+ | Gestión del clúster |
| Docker Engine | 26.1.4+ | Runtime de contenedores |

### Preparación Inicial

```bash
# Verificar que Minikube está en ejecución
minikube status

# Verificar acceso al clúster
kubectl cluster-info

# Verificar que el addon storage-provisioner está activo
minikube addons list | grep storage-provisioner

# Crear el directorio de trabajo del laboratorio
mkdir -p ~/kcna-labs/lab08
cd ~/kcna-labs/lab08
```

**Salida esperada** del addon storage-provisioner:
```
| storage-provisioner          | minikube | enabled ✅   |
```

---

## Paso a Paso

### Paso 1: Crear el Namespace y el Directorio hostPath

**Objetivo:** Preparar el namespace `lab08` y el directorio en el nodo Minikube que servirá como backend del PersistentVolume.

**Instrucciones:**

1. Crea el namespace `lab08`:

```bash
kubectl create namespace lab08
```

2. Crea el directorio hostPath dentro del nodo Minikube:

```bash
minikube ssh -- sudo mkdir -p /mnt/lab08-data
minikube ssh -- sudo chmod 777 /mnt/lab08-data
```

3. Verifica que el directorio existe:

```bash
minikube ssh -- ls -la /mnt/ | grep lab08
```

**Salida esperada:**
```
namespace/lab08 created
drwxrwxrwx  2 root root 4096 ... lab08-data
```

**Verificación:**
```bash
kubectl get namespaces | grep lab08
```

---

### Paso 2: Crear el PersistentVolume Manual

**Objetivo:** Definir un PersistentVolume de 1Gi con hostPath apuntando al directorio creado en el nodo.

**Instrucciones:**

1. Crea el archivo de manifiesto del PersistentVolume:

```bash
cat <<'EOF' > ~/kcna-labs/lab08/pv-hostpath-01.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath-01
  labels:
    type: local
    lab: lab08
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/lab08-data
    type: DirectoryOrCreate
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab08/pv-hostpath-01.yaml
```

3. Verifica el estado del PV:

```bash
kubectl get pv pv-hostpath-01
```

**Salida esperada:**
```
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-hostpath-01   1Gi        RWO            Retain           Available           manual                   5s
```

**Verificación:**

El campo `STATUS` debe mostrar `Available`, indicando que el PV está listo para ser reclamado.

```bash
kubectl describe pv pv-hostpath-01 | grep -E "Status|Source|Path"
```

---

### Paso 3: Crear el PersistentVolumeClaim para MySQL

**Objetivo:** Crear un PVC que se enlace al PV manual recién creado, solicitando 1Gi con AccessMode ReadWriteOnce.

**Instrucciones:**

1. Crea el manifiesto del PVC:

```bash
cat <<'EOF' > ~/kcna-labs/lab08/pvc-mysql-data.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-mysql-data
  namespace: lab08
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab08/pvc-mysql-data.yaml
```

3. Verifica el enlace PVC → PV:

```bash
kubectl get pvc pvc-mysql-data -n lab08
```

**Salida esperada:**
```
NAME             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-mysql-data   Bound    pv-hostpath-01   1Gi        RWO            manual         3s
```

4. Confirma que el PV ahora muestra estado `Bound`:

```bash
kubectl get pv pv-hostpath-01
```

**Salida esperada:**
```
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   AGE
pv-hostpath-01   1Gi        RWO            Retain           Bound    lab08/pvc-mysql-data   manual         45s
```

**Verificación:**

El PVC debe estar en estado `Bound` y el campo `VOLUME` debe mostrar `pv-hostpath-01`.

---

### Paso 4: Crear el Secret para MySQL

**Objetivo:** Crear un Secret con la contraseña root de MySQL que será consumida por el Pod como variable de entorno.

**Instrucciones:**

1. Crea el Secret de forma imperativa:

```bash
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=Lab08SecurePass! \
  -n lab08
```

2. Verifica que el Secret existe:

```bash
kubectl get secret mysql-secret -n lab08
```

**Salida esperada:**
```
NAME           TYPE     DATA   AGE
mysql-secret   Opaque   1      2s
```

3. (Opcional) Confirma el contenido codificado:

```bash
kubectl get secret mysql-secret -n lab08 -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
```

**Salida esperada:**
```
Lab08SecurePass!
```

---

### Paso 5: Desplegar el Pod MySQL con PVC

**Objetivo:** Crear un Pod estático de MySQL 8.3.0 que monte el PVC en `/var/lib/mysql` para persistir los datos de la base de datos.

**Instrucciones:**

1. Crea el manifiesto del Pod:

```bash
cat <<'EOF' > ~/kcna-labs/lab08/mysql-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  namespace: lab08
  labels:
    app: mysql
    lab: lab08
spec:
  containers:
    - name: mysql
      image: mysql:8.3.0
      ports:
        - containerPort: 3306
          name: mysql
      env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
      volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
  volumes:
    - name: mysql-persistent-storage
      persistentVolumeClaim:
        claimName: pvc-mysql-data
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab08/mysql-pod.yaml
```

3. Espera a que el Pod esté en estado `Running`:

```bash
kubectl wait --for=condition=Ready pod/mysql-pod -n lab08 --timeout=120s
```

4. Verifica el estado del Pod:

```bash
kubectl get pod mysql-pod -n lab08
```

**Salida esperada:**
```
NAME        READY   STATUS    RESTARTS   AGE
mysql-pod   1/1     Running   0          15s
```

5. Confirma que el volumen está montado correctamente:

```bash
kubectl describe pod mysql-pod -n lab08 | grep -A 3 "Mounts:"
```

**Salida esperada:**
```
    Mounts:
      /var/lib/mysql from mysql-persistent-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-... (ro)
```

**Verificación:**

```bash
kubectl exec mysql-pod -n lab08 -- df -h /var/lib/mysql
```

El punto de montaje debe aparecer con el tamaño aproximado del PV.

---

### Paso 6: Insertar Datos de Prueba en MySQL

**Objetivo:** Crear una base de datos y una tabla con un registro de prueba que permita verificar la persistencia después de eliminar el Pod.

**Instrucciones:**

1. Conéctate a MySQL y crea la base de datos de prueba:

```bash
kubectl exec -it mysql-pod -n lab08 -- mysql -u root -p'Lab08SecurePass!' -e "
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE registros (
  id INT AUTO_INCREMENT PRIMARY KEY,
  mensaje VARCHAR(255) NOT NULL,
  fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO registros (mensaje) VALUES ('Dato persistente del Lab 08');
SELECT * FROM registros;
"
```

**Salida esperada:**
```
+----+-----------------------------+---------------------+
| id | mensaje                     | fecha               |
+----+-----------------------------+---------------------+
|  1 | Dato persistente del Lab 08 | 2024-XX-XX XX:XX:XX |
+----+-----------------------------+---------------------+
```

2. Verifica que la base de datos existe:

```bash
kubectl exec mysql-pod -n lab08 -- mysql -u root -p'Lab08SecurePass!' -e "SHOW DATABASES;"
```

**Salida esperada (parcial):**
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| testdb             |
+--------------------+
```

**Verificación:**

La base de datos `testdb` debe aparecer en la lista y la tabla `registros` debe contener un registro.

---

### Paso 7: Verificar Persistencia Eliminando y Recreando el Pod

**Objetivo:** Demostrar que los datos sobreviven a la eliminación del Pod gracias al PVC.

**Instrucciones:**

1. Elimina el Pod de MySQL:

```bash
kubectl delete pod mysql-pod -n lab08
```

2. Confirma que el Pod fue eliminado:

```bash
kubectl get pods -n lab08
```

**Salida esperada:**
```
No resources found in lab08 namespace.
```

3. Verifica que el PVC sigue enlazado:

```bash
kubectl get pvc pvc-mysql-data -n lab08
```

**Salida esperada:**
```
NAME             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-mysql-data   Bound    pv-hostpath-01   1Gi        RWO            manual         5m
```

4. Verifica que los datos persisten en el nodo:

```bash
minikube ssh -- ls /mnt/lab08-data/ | head -5
```

Deberías ver archivos de MySQL como `ibdata1`, `ib_logfile0`, directorios de bases de datos, etc.

5. Recrea el Pod de MySQL con el mismo manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab08/mysql-pod.yaml
kubectl wait --for=condition=Ready pod/mysql-pod -n lab08 --timeout=120s
```

6. Consulta los datos insertados anteriormente:

```bash
kubectl exec mysql-pod -n lab08 -- mysql -u root -p'Lab08SecurePass!' -e "
USE testdb;
SELECT * FROM registros;
"
```

**Salida esperada:**
```
+----+-----------------------------+---------------------+
| id | mensaje                     | fecha               |
+----+-----------------------------+---------------------+
|  1 | Dato persistente del Lab 08 | 2024-XX-XX XX:XX:XX |
+----+-----------------------------+---------------------+
```

**Verificación:**

El registro insertado antes de eliminar el Pod sigue presente. Esto confirma que el PVC con hostPath mantiene los datos de forma independiente al ciclo de vida del Pod.

---

### Paso 8: Aprovisionamiento Dinámico con StorageClass

**Objetivo:** Crear un segundo PVC utilizando la StorageClass `standard` de Minikube para demostrar el aprovisionamiento dinámico de volúmenes.

**Instrucciones:**

1. Verifica las StorageClasses disponibles:

```bash
kubectl get storageclass
```

**Salida esperada:**
```
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  ...
```

2. Crea el PVC con aprovisionamiento dinámico:

```bash
cat <<'EOF' > ~/kcna-labs/lab08/pvc-dynamic-01.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic-01
  namespace: lab08
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF
```

3. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab08/pvc-dynamic-01.yaml
```

4. Verifica que el PVC se enlazó automáticamente a un PV creado dinámicamente:

```bash
kubectl get pvc pvc-dynamic-01 -n lab08
```

**Salida esperada:**
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-dynamic-01   Bound    pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   500Mi      RWO            standard       5s
```

5. Lista todos los PVs para ver el nuevo PV dinámico:

```bash
kubectl get pv
```

**Salida esperada (parcial):**
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS
pv-hostpath-01                             1Gi        RWO            Retain           Bound    lab08/pvc-mysql-data   manual
pvc-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX   500Mi      RWO            Delete           Bound    lab08/pvc-dynamic-01   standard
```

**Verificación:**

Observa las diferencias clave entre ambos PVs:
- El PV manual tiene `RECLAIM POLICY: Retain` y `STORAGECLASS: manual`
- El PV dinámico tiene `RECLAIM POLICY: Delete` y `STORAGECLASS: standard`

---

### Paso 9: Contrastar emptyDir vs PVC con Pod Writer

**Objetivo:** Desplegar un Pod que monte tanto el PVC dinámico como un volumen emptyDir para observar las diferencias de comportamiento entre almacenamiento persistente y efímero.

**Instrucciones:**

1. Crea el manifiesto del Pod writer:

```bash
cat <<'EOF' > ~/kcna-labs/lab08/writer-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer-pod
  namespace: lab08
  labels:
    app: writer
    lab: lab08
spec:
  containers:
    - name: writer
      image: busybox:1.36.1
      command:
        - sh
        - -c
        - |
          echo "Dato en PVC dinámico - $(date)" > /data-persistent/pvc-test.txt
          echo "Dato en emptyDir - $(date)" > /data-ephemeral/emptydir-test.txt
          echo "Escritura completada."
          sleep 3600
      volumeMounts:
        - name: persistent-vol
          mountPath: /data-persistent
        - name: ephemeral-vol
          mountPath: /data-ephemeral
  volumes:
    - name: persistent-vol
      persistentVolumeClaim:
        claimName: pvc-dynamic-01
    - name: ephemeral-vol
      emptyDir: {}
EOF
```

2. Aplica el manifiesto:

```bash
kubectl apply -f ~/kcna-labs/lab08/writer-pod.yaml
kubectl wait --for=condition=Ready pod/writer-pod -n lab08 --timeout=60s
```

3. Verifica que ambos archivos fueron escritos:

```bash
kubectl exec writer-pod -n lab08 -- cat /data-persistent/pvc-test.txt
kubectl exec writer-pod -n lab08 -- cat /data-ephemeral/emptydir-test.txt
```

**Salida esperada:**
```
Dato en PVC dinámico - Mon Jun XX XX:XX:XX UTC 2024
Dato en emptyDir - Mon Jun XX XX:XX:XX UTC 2024
```

4. Elimina el Pod writer:

```bash
kubectl delete pod writer-pod -n lab08
```

5. Recrea el Pod writer (con un comando de lectura):

```bash
cat <<'EOF' > ~/kcna-labs/lab08/reader-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
  namespace: lab08
  labels:
    app: reader
    lab: lab08
spec:
  containers:
    - name: reader
      image: busybox:1.36.1
      command:
        - sh
        - -c
        - |
          echo "=== Contenido PVC (persistente) ==="
          cat /data-persistent/pvc-test.txt 2>/dev/null || echo "ARCHIVO NO ENCONTRADO"
          echo ""
          echo "=== Contenido emptyDir (efímero) ==="
          cat /data-ephemeral/emptydir-test.txt 2>/dev/null || echo "ARCHIVO NO ENCONTRADO"
          sleep 3600
      volumeMounts:
        - name: persistent-vol
          mountPath: /data-persistent
        - name: ephemeral-vol
          mountPath: /data-ephemeral
  volumes:
    - name: persistent-vol
      persistentVolumeClaim:
        claimName: pvc-dynamic-01
    - name: ephemeral-vol
      emptyDir: {}
EOF
```

```bash
kubectl apply -f ~/kcna-labs/lab08/reader-pod.yaml
kubectl wait --for=condition=Ready pod/reader-pod -n lab08 --timeout=60s
```

6. Verifica los resultados:

```bash
kubectl logs reader-pod -n lab08
```

**Salida esperada:**
```
=== Contenido PVC (persistente) ===
Dato en PVC dinámico - Mon Jun XX XX:XX:XX UTC 2024

=== Contenido emptyDir (efímero) ===
ARCHIVO NO ENCONTRADO
```

**Verificación:**

El resultado demuestra claramente la diferencia:
- **PVC (persistente):** El archivo sobrevive a la eliminación del Pod
- **emptyDir (efímero):** El archivo desaparece cuando el Pod es eliminado

---

## Validación y Testing

Ejecuta las siguientes verificaciones para confirmar que el laboratorio se completó correctamente:

```bash
echo "=== Verificación Final del Lab 08 ==="
echo ""

echo "1. Namespace lab08 existe:"
kubectl get namespace lab08 -o name

echo ""
echo "2. PV pv-hostpath-01 está Bound:"
kubectl get pv pv-hostpath-01 -o jsonpath='{.status.phase}'
echo ""

echo ""
echo "3. PVC pvc-mysql-data está Bound:"
kubectl get pvc pvc-mysql-data -n lab08 -o jsonpath='{.status.phase}'
echo ""

echo ""
echo "4. PVC pvc-dynamic-01 está Bound:"
kubectl get pvc pvc-dynamic-01 -n lab08 -o jsonpath='{.status.phase}'
echo ""

echo ""
echo "5. Pod mysql-pod está Running:"
kubectl get pod mysql-pod -n lab08 -o jsonpath='{.status.phase}'
echo ""

echo ""
echo "6. Datos persisten en MySQL:"
kubectl exec mysql-pod -n lab08 -- mysql -u root -p'Lab08SecurePass!' -N -e "USE testdb; SELECT mensaje FROM registros WHERE id=1;"

echo ""
echo "7. Secret mysql-secret existe:"
kubectl get secret mysql-secret -n lab08 -o name

echo ""
echo "=== Verificación completada ==="
```

**Salida esperada completa:**
```
=== Verificación Final del Lab 08 ===

1. Namespace lab08 existe:
namespace/lab08

2. PV pv-hostpath-01 está Bound:
Bound

3. PVC pvc-mysql-data está Bound:
Bound

4. PVC pvc-dynamic-01 está Bound:
Bound

5. Pod mysql-pod está Running:
Running

6. Datos persisten en MySQL:
Dato persistente del Lab 08

7. Secret mysql-secret existe:
secret/mysql-secret

=== Verificación completada ===
```

---

## Solución de Problemas

### Problema 1: PVC permanece en estado Pending

**Síntomas:**
```
kubectl get pvc pvc-mysql-data -n lab08
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-mysql-data   Pending                                      manual         30s
```

**Causa:** El PVC no encuentra un PV compatible. Las causas más comunes son:
- El `storageClassName` del PVC no coincide con el del PV
- El `accessModes` del PVC no coincide con el del PV
- La capacidad solicitada en el PVC excede la capacidad del PV

**Solución:**

```bash
# Verificar eventos del PVC
kubectl describe pvc pvc-mysql-data -n lab08 | grep -A 5 "Events:"

# Comparar storageClassName entre PV y PVC
kubectl get pv pv-hostpath-01 -o jsonpath='{.spec.storageClassName}'
kubectl get pvc pvc-mysql-data -n lab08 -o jsonpath='{.spec.storageClassName}'

# Si no coinciden, elimina el PVC, corrige el YAML y vuelve a aplicar
kubectl delete pvc pvc-mysql-data -n lab08
# Editar el archivo pvc-mysql-data.yaml para que storageClassName sea "manual"
kubectl apply -f ~/kcna-labs/lab08/pvc-mysql-data.yaml
```

### Problema 2: Pod MySQL entra en CrashLoopBackOff

**Síntomas:**
```
kubectl get pod mysql-pod -n lab08
NAME        READY   STATUS             RESTARTS   AGE
mysql-pod   0/1     CrashLoopBackOff   3          2m
```

**Causa:** MySQL falla al inicializar porque:
- El directorio `/var/lib/mysql` contiene datos corruptos de un intento previo fallido
- La variable `MYSQL_ROOT_PASSWORD` no se inyecta correctamente desde el Secret
- El directorio hostPath no tiene permisos de escritura

**Solución:**

```bash
# Revisar logs del contenedor
kubectl logs mysql-pod -n lab08 --previous

# Si el error menciona permisos o datos corruptos, limpiar el hostPath
minikube ssh -- sudo rm -rf /mnt/lab08-data/*

# Verificar que el Secret existe y tiene la clave correcta
kubectl get secret mysql-secret -n lab08 -o jsonpath='{.data}' | jq .

# Verificar permisos del directorio
minikube ssh -- ls -la /mnt/lab08-data

# Si los permisos son incorrectos
minikube ssh -- sudo chmod 777 /mnt/lab08-data

# Eliminar y recrear el Pod
kubectl delete pod mysql-pod -n lab08
kubectl apply -f ~/kcna-labs/lab08/mysql-pod.yaml
```

---

## Limpieza

> **⚠️ IMPORTANTE:** El Pod `mysql-pod` y el PVC `pvc-mysql-data` serán utilizados intencionalmente en el Lab 09. **NO elimines** el namespace `lab08` ni sus recursos después de completar este laboratorio.

Si necesitas limpiar únicamente los Pods auxiliares de prueba:

```bash
# Eliminar solo los Pods de lectura/escritura auxiliares
kubectl delete pod reader-pod -n lab08 --ignore-not-found=true
kubectl delete pod writer-pod -n lab08 --ignore-not-found=true
```

Para verificar el estado final que debe permanecer:

```bash
kubectl get all,pvc,secret -n lab08
```

**Estado esperado que debe mantenerse:**
```
NAME            READY   STATUS    RESTARTS   AGE
pod/mysql-pod   1/1     Running   0          ...

NAME                             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/pvc-mysql-data    Bound    pv-hostpath-01   1Gi        RWO            manual
persistentvolumeclaim/pvc-dynamic-01    Bound    pvc-...          500Mi      RWO            standard

NAME                  TYPE     DATA   AGE
secret/mysql-secret   Opaque   1      ...
```

---

## Resumen

En este laboratorio has aplicado los conceptos fundamentales de almacenamiento persistente en Kubernetes:

| Concepto | Lo que aprendiste |
|----------|-------------------|
| **PersistentVolume (PV)** | Recurso de clúster que representa almacenamiento físico; creado manualmente con hostPath |
| **PersistentVolumeClaim (PVC)** | Solicitud de almacenamiento por parte de un Pod; se enlaza a un PV compatible |
| **StorageClass** | Permite aprovisionamiento dinámico; Minikube incluye `standard` por defecto |
| **AccessMode RWO** | ReadWriteOnce permite montaje de lectura/escritura en un solo nodo |
| **Reclaim Policy** | `Retain` conserva datos tras liberar el PVC; `Delete` los elimina |
| **emptyDir vs PVC** | emptyDir es efímero (muere con el Pod); PVC persiste independientemente |

### Conceptos Clave Demostrados

1. **Desacoplamiento almacenamiento-Pod:** El PVC existe independientemente del Pod, permitiendo recrear el Pod sin perder datos.
2. **Aprovisionamiento manual vs dinámico:** El PV manual requiere crear el recurso explícitamente; la StorageClass automatiza la creación del PV.
3. **Persistencia real verificada:** Los datos de MySQL sobrevivieron a la eliminación completa del Pod.

### Recursos Adicionales

- [Documentación oficial: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Documentación oficial: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Documentación oficial: Volumes - emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [MySQL Docker Hub - Configuración de variables de entorno](https://hub.docker.com/_/mysql)
