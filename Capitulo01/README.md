# 8 Práctica 1. Construcción y ejecución básica de contenedores

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio construirás desde cero una aplicación web Python Flask mínima, la empaquetarás en una imagen Docker optimizada siguiendo buenas prácticas OCI, ejecutarás contenedores localmente verificando el aislamiento de procesos y recursos, y finalmente publicarás la imagen en Docker Hub. Este artefacto publicado será consumido directamente en los laboratorios posteriores del curso.

## Objetivos de Aprendizaje

- [ ] Distinguir conceptualmente entre imagen, contenedor y máquina virtual identificando sus diferencias en recursos y aislamiento
- [ ] Construir una imagen Docker personalizada aplicando buenas prácticas de capas, tamaño mínimo y usuario no-root
- [ ] Ejecutar, inspeccionar, detener y eliminar contenedores usando comandos Docker fundamentales
- [ ] Analizar las capas de una imagen con `docker history` comprendiendo el sistema de archivos en capas (OverlayFS)
- [ ] Publicar una imagen en Docker Hub preparando el artefacto para su uso en laboratorios posteriores

## Prerrequisitos

### Conocimientos Previos

- Comprensión básica de qué es un contenedor, namespaces y cgroups (lección 1.1)
- Familiaridad con la línea de comandos Linux (navegación de directorios, edición de archivos)
- Conceptos básicos de aplicaciones web (peticiones HTTP, puertos)

### Acceso Requerido

- Docker Engine 26.1.4 instalado y funcionando (verificar con `docker run hello-world`)
- Cuenta activa en Docker Hub ([hub.docker.com](https://hub.docker.com))
- Conexión a Internet para descargar imágenes base y publicar la imagen final

## Entorno de Laboratorio

### Software Necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Docker Engine | 26.1.4 | Construcción y ejecución de contenedores |
| Python (imagen base) | 3.12.3-slim | Imagen base para la aplicación |
| curl | 8.8.0 | Verificación de endpoints HTTP |
| Git | 2.45.2 | Control de versiones del proyecto |
| VS Code (opcional) | 1.90.2 | Editor de código |

### Preparación Inicial del Entorno

```bash
# Verificar que Docker está funcionando correctamente
docker --version
docker run --rm hello-world

# Crear el directorio base del curso e inicializar Git
mkdir -p ~/kcna-labs/lab01
cd ~/kcna-labs
git init
echo "# KCNA Labs" > README.md
git add README.md
git commit -m "Inicializar repositorio de laboratorios KCNA"

# Posicionarse en el directorio del laboratorio
cd ~/kcna-labs/lab01
```

---

## Paso 1: Crear la Aplicación Web Flask

**Objetivo:** Escribir una aplicación Python Flask mínima que responda en el puerto 8080 con información JSON del host.

### Instrucciones

1. Crea el archivo de dependencias `requirements.txt`:

```bash
cat > ~/kcna-labs/lab01/requirements.txt << 'EOF'
flask==3.0.3
EOF
```

2. Crea el archivo principal de la aplicación `app.py`:

```bash
cat > ~/kcna-labs/lab01/app.py << 'EOF'
"""KCNA Webapp - Aplicación Flask mínima para laboratorios de contenedores."""

import os
import socket
from flask import Flask, jsonify

app = Flask(__name__)

VERSION = "1.0.0"
ENVIRONMENT = os.getenv("APP_ENVIRONMENT", "development")


@app.route("/")
def index():
    """Endpoint principal que retorna información del Pod/contenedor."""
    return jsonify({
        "hostname": socket.gethostname(),
        "version": VERSION,
        "environment": ENVIRONMENT,
        "message": "Hello from KCNA Webapp!"
    })


@app.route("/health")
def health():
    """Endpoint de salud para readiness/liveness probes."""
    return jsonify({"status": "healthy"}), 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
EOF
```

3. Verifica que ambos archivos existen:

```bash
ls -la ~/kcna-labs/lab01/
```

### Salida Esperada

```
total 16
drwxr-xr-x 2 user user 4096 ... .
drwxr-xr-x 3 user user 4096 ... ..
-rw-r--r-- 1 user user  623 ... app.py
-rw-r--r-- 1 user user   13 ... requirements.txt
```

### Verificación

```bash
# Confirmar que app.py tiene el puerto 8080 configurado
grep "port=8080" ~/kcna-labs/lab01/app.py
```

Debe mostrar: `app.run(host="0.0.0.0", port=8080)`

---

## Paso 2: Crear el Dockerfile Optimizado

**Objetivo:** Escribir un Dockerfile que siga buenas prácticas: imagen base slim, usuario no-root, orden de capas optimizado y etiquetas OCI.

### Instrucciones

1. Crea el archivo `Dockerfile`:

```bash
cat > ~/kcna-labs/lab01/Dockerfile << 'EOF'
# Etapa única: imagen de producción optimizada
FROM python:3.12.3-slim

# Etiquetas OCI estándar
LABEL org.opencontainers.image.title="kcna-webapp"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.description="Aplicación web Flask para laboratorios KCNA"
LABEL org.opencontainers.image.authors="estudiante-kcna"

# Evitar prompts interactivos y establecer encoding
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Crear usuario no-root para seguridad
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

# Establecer directorio de trabajo
WORKDIR /app

# Copiar dependencias primero (optimización de caché de capas)
COPY requirements.txt .

# Instalar dependencias sin caché de pip
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY app.py .

# Cambiar al usuario no-root
USER appuser

# Exponer el puerto de la aplicación (documentación)
EXPOSE 8080

# Comando de inicio
CMD ["python", "app.py"]
EOF
```

2. Crea el archivo `.dockerignore` para excluir archivos innecesarios:

```bash
cat > ~/kcna-labs/lab01/.dockerignore << 'EOF'
__pycache__
*.pyc
*.pyo
.git
.gitignore
.dockerignore
Dockerfile
README.md
*.md
.venv
.env
EOF
```

3. Revisa la estructura completa del proyecto:

```bash
ls -la ~/kcna-labs/lab01/
```

### Salida Esperada

```
total 24
drwxr-xr-x 2 user user 4096 ... .
drwxr-xr-x 3 user user 4096 ... ..
-rw-r--r-- 1 user user  112 ... .dockerignore
-rw-r--r-- 1 user user  623 ... app.py
-rw-r--r-- 1 user user  812 ... Dockerfile
-rw-r--r-- 1 user user   13 ... requirements.txt
```

### Verificación

```bash
# Verificar que el Dockerfile usa la imagen base correcta
head -2 ~/kcna-labs/lab01/Dockerfile | grep "python:3.12.3-slim"

# Verificar que se usa usuario no-root
grep "^USER appuser" ~/kcna-labs/lab01/Dockerfile
```

> **Nota sobre buenas prácticas aplicadas:**
> - **Imagen base slim**: `python:3.12.3-slim` (~120 MB) en lugar de `python:3.12.3` (~900 MB)
> - **Orden de capas**: `requirements.txt` se copia antes que `app.py` para aprovechar la caché de Docker cuando solo cambia el código
> - **Usuario no-root**: Reduce el impacto de una posible vulnerabilidad dentro del contenedor
> - **`.dockerignore`**: Evita enviar archivos innecesarios al contexto de build

---

## Paso 3: Construir la Imagen Docker

**Objetivo:** Construir la imagen Docker localmente con el tag `kcna-webapp:1.0.0` y verificar su creación.

### Instrucciones

1. Posiciónate en el directorio del laboratorio y construye la imagen:

```bash
cd ~/kcna-labs/lab01
docker build -t kcna-webapp:1.0.0 .
```

2. Verifica que la imagen fue creada exitosamente:

```bash
docker images kcna-webapp
```

3. Inspecciona el tamaño de la imagen y compárala con la imagen base:

```bash
docker images python:3.12.3-slim
docker images kcna-webapp:1.0.0
```

### Salida Esperada

La construcción debe completarse sin errores. La salida del build mostrará cada capa:

```
[+] Building 15.2s (11/11) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [internal] load metadata for docker.io/library/python:3.12.3-slim
 => [1/6] FROM docker.io/library/python:3.12.3-slim@sha256:...
 => [2/6] RUN groupadd -r appuser && useradd ...
 => [3/6] WORKDIR /app
 => [4/6] COPY requirements.txt .
 => [5/6] RUN pip install --no-cache-dir -r requirements.txt
 => [6/6] COPY app.py .
 => exporting to image
 => => naming to docker.io/library/kcna-webapp:1.0.0
```

La tabla de imágenes mostrará algo similar a:

```
REPOSITORY    TAG     IMAGE ID       CREATED          SIZE
kcna-webapp   1.0.0   a1b2c3d4e5f6   10 seconds ago   145MB
```

### Verificación

```bash
# Confirmar que la imagen existe con el tag correcto
docker image inspect kcna-webapp:1.0.0 --format '{{.Config.Labels}}' | grep "1.0.0"
```

---

## Paso 4: Analizar las Capas de la Imagen

**Objetivo:** Comprender el sistema de archivos en capas examinando la estructura interna de la imagen con `docker history`.

### Instrucciones

1. Examina las capas de la imagen:

```bash
docker history kcna-webapp:1.0.0
```

2. Obtén una vista más detallada sin truncar los comandos:

```bash
docker history --no-trunc --format "table {{.CreatedBy}}\t{{.Size}}" kcna-webapp:1.0.0
```

3. Inspecciona los metadatos de la imagen:

```bash
docker image inspect kcna-webapp:1.0.0 --format '{{json .Config}}' | python3 -m json.tool
```

### Salida Esperada

```
IMAGE          CREATED          CREATED BY                                      SIZE
a1b2c3d4e5f6   1 minute ago     CMD ["python" "app.py"]                         0B
<missing>      1 minute ago     EXPOSE map[8080/tcp:{}]                         0B
<missing>      1 minute ago     USER appuser                                    0B
<missing>      1 minute ago     COPY app.py . # buildkit                        623B
<missing>      1 minute ago     RUN pip install --no-cache-dir -r requi...       12.5MB
<missing>      1 minute ago     COPY requirements.txt . # buildkit              13B
<missing>      1 minute ago     WORKDIR /app                                    0B
<missing>      1 minute ago     RUN groupadd -r appuser && useradd ...          8.8kB
<missing>      2 weeks ago      /bin/sh -c #(nop)  ENV PYTHON_VERSION=3...      0B
...
```

### Verificación

```bash
# Contar el número de capas de la imagen
docker image inspect kcna-webapp:1.0.0 --format '{{len .RootFS.Layers}}' 
```

El resultado debe ser un número entre 6 y 10 capas (las de la base más las añadidas por nuestro Dockerfile).

> **Concepto clave:** Cada instrucción `RUN`, `COPY` o `ADD` en el Dockerfile crea una nueva capa de solo lectura. Las capas se apilan usando OverlayFS. Cuando ejecutas un contenedor, se añade una capa de escritura temporal encima. Esto es lo que hace a las imágenes compartibles y eficientes en disco.

---

## Paso 5: Ejecutar el Contenedor Localmente

**Objetivo:** Ejecutar la imagen como contenedor, verificar su funcionamiento con `curl` y observar el aislamiento de procesos.

### Instrucciones

1. Ejecuta el contenedor en modo detached (segundo plano) mapeando el puerto 8080:

```bash
docker run -d --name kcna-webapp-test -p 8080:8080 kcna-webapp:1.0.0
```

2. Verifica que el contenedor está corriendo:

```bash
docker ps --filter "name=kcna-webapp-test"
```

3. Prueba el endpoint principal:

```bash
curl -s http://localhost:8080/ | python3 -m json.tool
```

4. Prueba el endpoint de salud:

```bash
curl -s http://localhost:8080/health | python3 -m json.tool
```

5. Observa los logs del contenedor:

```bash
docker logs kcna-webapp-test
```

### Salida Esperada

Respuesta del endpoint principal:

```json
{
    "hostname": "a1b2c3d4e5f6",
    "version": "1.0.0",
    "environment": "development",
    "message": "Hello from KCNA Webapp!"
}
```

Respuesta del endpoint de salud:

```json
{
    "status": "healthy"
}
```

Logs del contenedor:

```
 * Serving Flask app 'app'
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:8080
 * Running on http://172.17.0.2:8080
```

### Verificación

```bash
# Verificar el código de respuesta HTTP
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health
```

Debe retornar: `200`

---

## Paso 6: Inspeccionar el Contenedor y Verificar el Aislamiento

**Objetivo:** Explorar el contenedor en ejecución para comprender el aislamiento de procesos, red y sistema de archivos proporcionado por namespaces y cgroups.

### Instrucciones

1. Inspecciona los detalles del contenedor:

```bash
docker inspect kcna-webapp-test --format '{{.State.Pid}}'
```

2. Verifica que el contenedor ejecuta como usuario no-root:

```bash
docker exec kcna-webapp-test whoami
```

3. Lista los procesos dentro del contenedor (aislamiento por namespace `pid`):

```bash
docker exec kcna-webapp-test ps aux
```

4. Verifica el aislamiento de red (namespace `net`):

```bash
docker exec kcna-webapp-test hostname
docker exec kcna-webapp-test cat /etc/hosts
```

5. Verifica el sistema de archivos aislado (namespace `mnt`):

```bash
docker exec kcna-webapp-test ls /app
```

6. Comprueba los límites de recursos desde el host:

```bash
docker stats --no-stream kcna-webapp-test
```

### Salida Esperada

Comando `whoami`:
```
appuser
```

Comando `ps aux` (solo procesos del contenedor visible):
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
appuser      1  0.1  0.5  33456 23456 ?        Ss   10:00   0:00 python app.py
appuser     12  0.0  0.0   6456  2340 ?        Rs   10:01   0:00 ps aux
```

Comando `ls /app`:
```
app.py
requirements.txt
```

### Verificación

```bash
# Confirmar que el PID 1 del contenedor es nuestro proceso Python
docker exec kcna-webapp-test cat /proc/1/cmdline | tr '\0' ' '
```

Debe mostrar: `python app.py`

> **Relación con la teoría:** Lo que observas aquí es el resultado directo de los **namespaces** del kernel Linux que estudiamos en la lección 1.1. El contenedor tiene su propio namespace `pid` (solo ve sus procesos), su propio namespace `net` (hostname propio) y su propio namespace `mnt` (solo ve `/app` con nuestros archivos). Los **cgroups** se reflejan en los límites que muestra `docker stats`.

---

## Paso 7: Comparar Contenedor vs Máquina Virtual

**Objetivo:** Entender visualmente las diferencias fundamentales entre un contenedor y una máquina virtual en términos de recursos y arranque.

### Instrucciones

1. Mide el tiempo de arranque de un contenedor:

```bash
time docker run --rm kcna-webapp:1.0.0 python -c "print('Contenedor listo')"
```

2. Verifica el uso de memoria del contenedor en ejecución:

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}" kcna-webapp-test
```

3. Compara el tamaño de nuestra imagen con una imagen de sistema operativo completo:

```bash
# Nuestra imagen optimizada
docker images kcna-webapp:1.0.0 --format "{{.Repository}}:{{.Tag}} - {{.Size}}"

# Una imagen de Ubuntu completa (para comparación)
docker pull ubuntu:22.04 2>/dev/null
docker images ubuntu:22.04 --format "{{.Repository}}:{{.Tag}} - {{.Size}}"
```

### Salida Esperada

Tiempo de arranque del contenedor:
```
Contenedor listo

real    0m0.8s
user    0m0.0s
sys     0m0.0s
```

Uso de memoria:
```
NAME                MEM USAGE / LIMIT     CPU %
kcna-webapp-test    28.5MiB / 15.63GiB    0.02%
```

Comparación de tamaños:
```
kcna-webapp:1.0.0 - 145MB
ubuntu:22.04 - 77.9MB
```

### Verificación

La siguiente tabla resume las diferencias clave observadas:

| Característica | Contenedor | Máquina Virtual |
|---|---|---|
| **Tiempo de arranque** | < 1 segundo | 30-60 segundos |
| **Uso de RAM** | ~30 MB (solo la app) | 512 MB - 2 GB (SO completo) |
| **Tamaño en disco** | ~145 MB (imagen) | 5-20 GB (disco virtual) |
| **Kernel** | Compartido con el host | Propio (virtualizado) |
| **Aislamiento** | Namespaces + cgroups | Hypervisor completo |
| **Densidad** | Cientos por host | Decenas por host |

> **Concepto clave:** Un contenedor comparte el kernel del host y solo aísla procesos mediante namespaces/cgroups. Una VM incluye su propio kernel y sistema operativo completo virtualizado por un hypervisor. Esto explica la diferencia dramática en recursos y velocidad de arranque.

---

## Paso 8: Detener, Eliminar y Gestionar el Ciclo de Vida

**Objetivo:** Practicar los comandos fundamentales del ciclo de vida de contenedores: stop, start, rm.

### Instrucciones

1. Detén el contenedor en ejecución:

```bash
docker stop kcna-webapp-test
```

2. Verifica que el contenedor está detenido:

```bash
docker ps -a --filter "name=kcna-webapp-test"
```

3. Reinicia el contenedor detenido:

```bash
docker start kcna-webapp-test
```

4. Verifica que responde nuevamente:

```bash
curl -s http://localhost:8080/ | python3 -m json.tool
```

5. Detén y elimina el contenedor:

```bash
docker stop kcna-webapp-test
docker rm kcna-webapp-test
```

6. Confirma que fue eliminado:

```bash
docker ps -a --filter "name=kcna-webapp-test"
```

### Salida Esperada

Después de `docker ps -a` con el contenedor detenido:
```
CONTAINER ID   IMAGE              COMMAND          STATUS                     NAMES
a1b2c3d4e5f6   kcna-webapp:1.0.0  "python app.py"  Exited (0) 5 seconds ago   kcna-webapp-test
```

Después de `docker rm`, el filtro no debe mostrar resultados:
```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

### Verificación

```bash
# Confirmar que no queda ningún contenedor con ese nombre
docker ps -a --filter "name=kcna-webapp-test" --format "{{.Names}}" | wc -l
```

Debe retornar: `0`

---

## Paso 9: Publicar la Imagen en Docker Hub

**Objetivo:** Etiquetar y publicar la imagen en Docker Hub para que esté disponible en los laboratorios posteriores.

### Instrucciones

1. Inicia sesión en Docker Hub (reemplaza `<tu-usuario-dockerhub>` con tu nombre de usuario real):

```bash
export DOCKERHUB_USER=<tu-usuario-dockerhub>
docker login -u $DOCKERHUB_USER
```

2. Etiqueta la imagen local con el nombre completo del registry:

```bash
docker tag kcna-webapp:1.0.0 ${DOCKERHUB_USER}/kcna-webapp:1.0.0
```

3. Verifica que la nueva etiqueta existe:

```bash
docker images | grep kcna-webapp
```

4. Publica la imagen en Docker Hub:

```bash
docker push ${DOCKERHUB_USER}/kcna-webapp:1.0.0
```

5. Verifica la publicación descargando la imagen desde el registry (simulando otro entorno):

```bash
# Eliminar la imagen local para forzar la descarga
docker rmi ${DOCKERHUB_USER}/kcna-webapp:1.0.0

# Descargar desde Docker Hub
docker pull ${DOCKERHUB_USER}/kcna-webapp:1.0.0
```

6. Ejecuta un contenedor con la imagen descargada para confirmar su integridad:

```bash
docker run --rm -p 8080:8080 ${DOCKERHUB_USER}/kcna-webapp:1.0.0 &
sleep 2
curl -s http://localhost:8080/ | python3 -m json.tool
docker stop $(docker ps -q --filter "ancestor=${DOCKERHUB_USER}/kcna-webapp:1.0.0")
```

### Salida Esperada

Push exitoso:
```
The push refers to repository [docker.io/<tu-usuario>/kcna-webapp]
5f70bf18a086: Pushed
a3ed95caeb02: Pushed
...
1.0.0: digest: sha256:abc123... size: 1987
```

Respuesta del contenedor descargado:
```json
{
    "hostname": "b2c3d4e5f6a7",
    "version": "1.0.0",
    "environment": "development",
    "message": "Hello from KCNA Webapp!"
}
```

### Verificación

```bash
# Verificar que la imagen está disponible en el registry
docker manifest inspect ${DOCKERHUB_USER}/kcna-webapp:1.0.0 > /dev/null 2>&1 && echo "✓ Imagen publicada correctamente en Docker Hub" || echo "✗ Error: imagen no encontrada en Docker Hub"
```

> **Importante:** Anota tu Docker Hub username (`$DOCKERHUB_USER`). Lo necesitarás en el laboratorio 02-00-01 para referenciar esta imagen en manifiestos de Kubernetes.

---

## Paso 10: Confirmar Cambios en Git

**Objetivo:** Versionar todos los archivos del laboratorio en el repositorio Git del curso.

### Instrucciones

1. Guarda tu Docker Hub username en un archivo de referencia:

```bash
echo "DOCKERHUB_USER=${DOCKERHUB_USER}" > ~/kcna-labs/lab01/.env
echo ".env" >> ~/kcna-labs/lab01/.dockerignore
```

2. Añade todos los archivos al repositorio y crea un commit:

```bash
cd ~/kcna-labs
cat > .gitignore << 'EOF'
.env
__pycache__/
*.pyc
EOF

git add .
git commit -m "Lab 01: Aplicación Flask + Dockerfile optimizado + imagen publicada en Docker Hub"
```

3. Verifica el estado del repositorio:

```bash
git log --oneline
```

### Salida Esperada

```
a1b2c3d Lab 01: Aplicación Flask + Dockerfile optimizado + imagen publicada en Docker Hub
f6e5d4c Inicializar repositorio de laboratorios KCNA
```

---

## Validación y Pruebas Finales

Ejecuta la siguiente secuencia completa de verificaciones para confirmar que el laboratorio se completó exitosamente:

```bash
echo "=== Validación Final del Lab 01 ==="
echo ""

# 1. Verificar que la imagen local existe
echo "1. Imagen local:"
docker images kcna-webapp:1.0.0 --format "   ✓ {{.Repository}}:{{.Tag}} ({{.Size}})" || echo "   ✗ Imagen local no encontrada"

# 2. Verificar que la imagen remota existe
echo "2. Imagen en Docker Hub:"
docker images ${DOCKERHUB_USER}/kcna-webapp:1.0.0 --format "   ✓ {{.Repository}}:{{.Tag}}" || echo "   ✗ Imagen remota no encontrada"

# 3. Ejecutar contenedor de prueba
echo "3. Ejecución de contenedor:"
docker run --rm -d --name validation-test -p 8080:8080 kcna-webapp:1.0.0 > /dev/null 2>&1
sleep 2

RESPONSE=$(curl -s http://localhost:8080/)
if echo "$RESPONSE" | grep -q '"version": "1.0.0"'; then
    echo "   ✓ Endpoint / responde correctamente"
else
    echo "   ✗ Endpoint / no responde como se espera"
fi

HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)
if [ "$HEALTH" = "200" ]; then
    echo "   ✓ Endpoint /health retorna HTTP 200"
else
    echo "   ✗ Endpoint /health no retorna HTTP 200"
fi

# 4. Verificar usuario no-root
USER_CHECK=$(docker exec validation-test whoami)
if [ "$USER_CHECK" = "appuser" ]; then
    echo "   ✓ Contenedor ejecuta como usuario no-root (appuser)"
else
    echo "   ✗ Contenedor NO ejecuta como usuario no-root"
fi

# 5. Limpieza del contenedor de validación
docker stop validation-test > /dev/null 2>&1

# 6. Verificar archivos en Git
echo "4. Repositorio Git:"
cd ~/kcna-labs
if git log --oneline | grep -q "Lab 01"; then
    echo "   ✓ Commit del Lab 01 presente"
else
    echo "   ✗ Commit del Lab 01 no encontrado"
fi

echo ""
echo "=== Validación Completa ==="
```

Resultado esperado: todos los checks deben mostrar `✓`.

---

## Solución de Problemas

### Problema 1: Error "permission denied" al ejecutar el contenedor

**Síntomas:**
```
PermissionError: [Errno 13] Permission denied: '/app/app.py'
```
O el contenedor se detiene inmediatamente con exit code 1.

**Causa:** El orden de las instrucciones en el Dockerfile es incorrecto. Si `USER appuser` se coloca antes de `COPY app.py .`, el archivo puede copiarse con permisos que el usuario `appuser` no puede leer, o el directorio `/app` no tiene los permisos adecuados.

**Solución:**

```bash
# Verificar el orden en el Dockerfile: USER debe estar DESPUÉS de todos los COPY y RUN
grep -n "USER\|COPY\|RUN" ~/kcna-labs/lab01/Dockerfile

# El orden correcto es:
# COPY requirements.txt .
# RUN pip install ...
# COPY app.py .
# USER appuser    <-- DESPUÉS de los COPY
# CMD [...]

# Si es necesario corregir, reconstruir la imagen:
cd ~/kcna-labs/lab01
docker build -t kcna-webapp:1.0.0 --no-cache .
```

---

### Problema 2: Error "port is already allocated" al ejecutar el contenedor

**Síntomas:**
```
docker: Error response from daemon: driver failed programming external connectivity:
Bind for 0.0.0.0:8080 failed: port is already allocated.
```

**Causa:** Otro proceso o contenedor ya está usando el puerto 8080 en el host. Puede ser un contenedor previo que no se detuvo correctamente o un servicio local.

**Solución:**

```bash
# Identificar qué está usando el puerto 8080
sudo lsof -i :8080
# O con ss:
ss -tlnp | grep 8080

# Si es un contenedor Docker anterior:
docker ps --filter "publish=8080"
docker stop $(docker ps -q --filter "publish=8080")

# Si es un proceso del sistema, detenerlo o usar otro puerto:
docker run -d --name kcna-webapp-test -p 9080:8080 kcna-webapp:1.0.0
# Luego acceder con: curl http://localhost:9080/
```

---

## Limpieza

Ejecuta los siguientes comandos para eliminar los recursos creados durante el laboratorio (mantén la imagen si vas a continuar con el Lab 02):

```bash
# Detener y eliminar contenedores del laboratorio
docker stop kcna-webapp-test 2>/dev/null
docker rm kcna-webapp-test 2>/dev/null

# Eliminar la imagen de Ubuntu usada para comparación
docker rmi ubuntu:22.04 2>/dev/null

# (OPCIONAL) Si NO vas a continuar con Lab 02, eliminar las imágenes:
# docker rmi kcna-webapp:1.0.0
# docker rmi ${DOCKERHUB_USER}/kcna-webapp:1.0.0

# Limpiar imágenes huérfanas (dangling)
docker image prune -f
```

> **Nota:** NO elimines la imagen `kcna-webapp:1.0.0` ni `${DOCKERHUB_USER}/kcna-webapp:1.0.0` si planeas continuar con el laboratorio 02-00-01, donde se usará como base para el despliegue en Kubernetes.

---

## Resumen

En este laboratorio has completado el ciclo completo de contenedorización de una aplicación:

| Actividad | Resultado |
|-----------|-----------|
| Crear aplicación Flask | `app.py` con endpoints `/` y `/health` en puerto 8080 |
| Escribir Dockerfile optimizado | Imagen base slim, usuario no-root, capas ordenadas, etiquetas OCI |
| Construir imagen | `kcna-webapp:1.0.0` (~145 MB) |
| Analizar capas | Comprensión del sistema de archivos OverlayFS |
| Ejecutar e inspeccionar | Verificación del aislamiento por namespaces y cgroups |
| Comparar con VMs | Diferencias en arranque (<1s vs 30-60s), memoria y tamaño |
| Publicar en Docker Hub | `<tu-usuario>/kcna-webapp:1.0.0` disponible públicamente |

### Conceptos Clave Reforzados

- **Imagen vs Contenedor**: La imagen es la plantilla inmutable (capas de solo lectura); el contenedor es una instancia en ejecución con una capa de escritura temporal encima.
- **Namespaces**: Proporcionan aislamiento de procesos (`pid`), red (`net`), sistema de archivos (`mnt`) y hostname (`uts`).
- **Cgroups**: Limitan el consumo de CPU y memoria de cada contenedor.
- **OCI**: El estándar Open Container Initiative define el formato de imágenes y el runtime, permitiendo interoperabilidad entre Docker, containerd, CRI-O y Podman.

### Preparación para el Siguiente Laboratorio

El laboratorio 02-00-01 utilizará la imagen `${DOCKERHUB_USER}/kcna-webapp:1.0.0` que acabas de publicar para desplegarla en un clúster Kubernetes local con Minikube. Asegúrate de tener anotado tu Docker Hub username.

### Recursos Adicionales

- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Docker CLI Reference](https://docs.docker.com/reference/cli/docker/)
- [CNCF Cloud Native Glossary](https://glossary.cncf.io/es/)
