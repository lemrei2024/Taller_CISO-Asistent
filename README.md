# Taller 1. Instalación de CISO Assistant en Kali Linux

**Curso:** Gobernanza en Seguridad de la Información  
**Tipo:** Taller guiado  

| Modalidad | Dedicación estimada | Entorno | Resultado |
|---|---|---|---|
| Individual | 2 a 3 horas | Kali Linux en máquina virtual | CISO Assistant funcionando en `https://localhost:8443` |

## Contenido

- [1. Objetivo](#1-objetivo)
- [2. Conceptos básicos](#2-conceptos-básicos)
- [3. Requisitos](#3-requisitos)
- [4. Instalación paso a paso](#4-instalación-paso-a-paso)
- [5. Solución de problemas](#5-solución-de-problemas)
- [6. Fuentes](#6-fuentes)

## 1. Objetivo

Instalar y dejar en funcionamiento CISO Assistant Community Edition, una plataforma de gobierno, riesgo y cumplimiento (GRC) de código abierto, sobre Kali Linux mediante contenedores Docker.

Al finalizar el taller, el estudiante estará en capacidad de:

1. Preparar una máquina virtual con Kali Linux con los recursos necesarios para ejecutar servicios en contenedores.
2. Instalar Docker Engine y Docker Compose desde el repositorio oficial de Docker.
3. Desplegar CISO Assistant con Docker Compose y verificar el estado de sus servicios.
4. Diagnosticar y resolver problemas comunes de despliegue a partir de los registros de los contenedores.

## 2. Conceptos básicos

| Concepto | Descripción |
|---|---|
| CISO Assistant | Plataforma GRC desarrollada por intuitem. Su edición comunitaria es de código abierto (licencia AGPLv3) e incluye bibliotecas con más de 200 marcos de referencia, entre ellos ISO/IEC 27001:2022 y NIST CSF 2.0. |
| Contenedor | Unidad que empaqueta una aplicación con todo lo que necesita para ejecutarse, aislada del resto del sistema. |
| Imagen | Plantilla de solo lectura a partir de la cual se crean los contenedores. CISO Assistant publica sus imágenes en el registro de GitHub (ghcr.io). |
| Docker Engine | Servicio que ejecuta los contenedores en el sistema operativo. |
| Docker Compose | Herramienta que define y ejecuta en conjunto varios contenedores relacionados a partir de un archivo `docker-compose.yml`. |
| Servicios de CISO Assistant | `backend` (lógica y base de datos), `frontend` (interfaz web), `caddy` (proxy con HTTPS en el puerto 8443), `huey` (tareas programadas) y `qdrant` (base de datos de apoyo). |
| Healthcheck | Verificación periódica que hace Docker para saber si un servicio está listo. Los servicios `frontend`, `caddy` y `huey` solo arrancan cuando el `backend` está en estado `healthy`. |

## 3. Requisitos

El taller se desarrolla sobre **Kali Linux**. CISO Assistant se ejecuta en contenedores de Docker dentro de Kali y se usa desde el navegador Firefox de Kali.

| Requisito | Detalle |
|---|---|
| Sistema operativo | Kali Linux actualizado (versión 2026.x), instalado en máquina virtual (VirtualBox o VMware) o directamente en el equipo. |
| Recursos de la máquina virtual | Mínimo 2 procesadores (vCPU), 6 GB de memoria RAM (se recomiendan 8 GB) y 40 GB de disco. |
| Conexión a internet | Necesaria para descargar los paquetes y las imágenes de la herramienta (aproximadamente 3 GB). |
| Usuario | Usuario de Kali con permisos de `sudo` (en las máquinas virtuales oficiales, el usuario por defecto es `kali`). |
| Docker | Docker Engine 27.0 o superior con el complemento Docker Compose v2. Se instala en la Parte 0. |

### 3.1 Configuración recomendada de la máquina virtual

Antes de iniciar Kali, en la configuración de la máquina virtual:

1. **Memoria:** asignar al menos 6144 MB (8192 MB si el equipo anfitrión tiene 16 GB o más).
2. **Procesadores:** asignar al menos 2.
3. **Disco:** verificar que el disco virtual tenga al menos 40 GB. Si la máquina virtual se descargó con un disco menor, ampliarlo antes de empezar.
4. **Red:** usar el modo **NAT**, que permite a Kali salir a internet.

**Alternativa si el equipo no tiene los recursos suficientes:** el estudiante debe informar al docente en la primera semana para trabajar en la instancia que el docente habilite para el curso, o solicitar la prueba gratuita en la nube que ofrece el fabricante en https://intuitem.com/trial. 

## 4. Instalación paso a paso

**Resultado esperado:** CISO Assistant funcionando en `https://localhost:8443` dentro de Kali Linux, con un usuario administrador creado.

**Tiempo estimado:** entre 40 y 90 minutos, según la conexión a internet.

> **Convenciones de esta guía.** Los comandos se escriben en la **terminal de Kali** (menú de aplicaciones, opción *Terminal Emulator*, o atajo `Ctrl + Alt + T`). Cada bloque se copia y se pega completo; para pegar en la terminal se usa `Ctrl + Shift + V`. Cuando un comando empieza por `sudo`, Kali solicita la contraseña del usuario; al escribirla no se muestran caracteres en pantalla, lo cual es normal.

### Paso 1. Actualizar Kali Linux

Kali es una distribución de actualización continua. Antes de instalar cualquier paquete se debe actualizar el sistema:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Si la actualización incluyó un nuevo kernel, reiniciar la máquina virtual:

```bash
sudo reboot
```

Verificar la versión de Kali:

```bash
cat /etc/os-release | head -n 4
```

### Paso 2. Instalar las herramientas básicas

```bash
sudo apt install -y ca-certificates curl gnupg git
```

| Paquete | Para qué se usa |
|---|---|
| `ca-certificates` | Validar los certificados de los sitios de descarga. |
| `curl` | Descargar la llave de firma del repositorio de Docker. |
| `gnupg` | Procesar la llave de firma. |
| `git` | Descargar el repositorio de CISO Assistant. |

### Paso 3. Retirar versiones de Docker que puedan generar conflicto

Kali incluye en sus repositorios un paquete llamado `docker.io`, cuya versión puede ser inferior a la que exige CISO Assistant. Además, el paquete llamado `docker` en Kali **no** corresponde a Docker. Para evitar conflictos, se retiran estas versiones si estuvieran instaladas:

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt remove -y $pkg
done
```

Es normal que el sistema indique que algunos de estos paquetes no están instalados.

### Paso 4. Agregar el repositorio oficial de Docker

Kali está basado en Debian, por lo que se usa el repositorio de Docker para la versión estable de Debian vigente, llamada **trixie**.

1. Crear la carpeta de llaves y descargar la llave de firma de Docker:

   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```

2. Registrar el repositorio:

   ```bash
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian trixie stable" | sudo tee /etc/apt/sources.list.d/docker.list
   ```

3. Actualizar la lista de paquetes:

   ```bash
   sudo apt update
   ```

   En la salida debe aparecer una línea que contenga `download.docker.com/linux/debian trixie`.

> **Nota.** Si en el futuro Debian publica una nueva versión estable, la documentación de Kali indicará el nombre que debe reemplazar a `trixie`.

### Paso 5. Instalar Docker Engine y Docker Compose

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Paquete | Función |
|---|---|
| `docker-ce` | Motor de Docker. |
| `docker-ce-cli` | Comandos de Docker para la terminal. |
| `containerd.io` | Entorno de ejecución de los contenedores. |
| `docker-buildx-plugin` | Construcción de imágenes. |
| `docker-compose-plugin` | Docker Compose v2, que permite ejecutar el comando `docker compose`. |

### Paso 6. Activar Docker y configurar el usuario

1. Iniciar el servicio de Docker y dejarlo activo en cada arranque de Kali:

   ```bash
   sudo systemctl enable docker --now
   ```

2. Verificar que el servicio está en ejecución:

   ```bash
   sudo systemctl status docker --no-pager
   ```

   La salida debe mostrar `Active: active (running)`.

3. Agregar el usuario actual al grupo `docker`, para ejecutar Docker sin `sudo`:

   ```bash
   sudo usermod -aG docker $USER
   ```

4. **Cerrar la sesión de Kali y volver a iniciarla** (o reiniciar la máquina virtual) para que el cambio de grupo tenga efecto.

5. Verificar la instalación:

   ```bash
   docker --version
   docker compose version
   docker run --rm hello-world
   ```

   Resultado esperado:

   - `docker --version` debe mostrar una versión **27.0 o superior**.
   - `docker compose version` debe mostrar Docker Compose **v2**.
   - `hello-world` debe mostrar el mensaje *Hello from Docker!*.

   Si `docker run` muestra un error de permisos (*permission denied*), el cambio de grupo aún no tiene efecto: cerrar la sesión y volver a ingresar.

### Paso 7. Descargar e iniciar CISO Assistant

1. Ubicarse en la carpeta personal y descargar el repositorio oficial:

   ```bash
   cd ~
   git clone --single-branch -b main https://github.com/intuitem/ciso-assistant-community.git
   cd ciso-assistant-community
   ```

2. Ejecutar el script de inicio para Linux:

   ```bash
   chmod +x docker-compose.sh
   ./docker-compose.sh
   ```

   El script descarga las imágenes de la herramienta, crea los contenedores e inicializa la base de datos. La primera ejecución puede tardar entre 5 y 20 minutos.

3. Cuando el script lo solicite, crear el **usuario administrador**:

   - **Correo electrónico:** el correo con el que se iniciará sesión.
   - **Contraseña:** una contraseña robusta, que se debe guardar.

4. Verificar que los contenedores están en ejecución:

   ```bash
   docker compose ps
   ```

   Todos los servicios deben aparecer con estado `running` o `Up`.

### Paso 7A. Si el primer arranque termina con "container backend is unhealthy"

En el primer arranque, el backend aplica las migraciones de la base de datos y carga más de 200 bibliotecas de marcos de referencia antes de responder. En máquinas virtuales con pocos recursos esto puede superar el tiempo de espera del script, que se detiene con el mensaje `dependency failed to start: container backend is unhealthy` **antes de solicitar el correo y la contraseña del administrador**. En la mayoría de los casos no es una falla: el backend continúa cargando en segundo plano.

1. Seguir el avance del backend:

   ```bash
   cd ~/ciso-assistant-community
   docker compose logs -f backend
   ```

   Esperar a que aparezcan las líneas de `gunicorn` con el texto `Listening at: http://0.0.0.0:8000`. Para salir del seguimiento: `Ctrl + C`.

2. Iniciar los demás servicios y verificar que el backend aparezca como `healthy`:

   ```bash
   docker compose up -d
   docker compose ps
   ```

3. Crear el usuario administrador manualmente (el script no alcanzó a hacerlo):

   ```bash
   docker compose exec backend python manage.py createsuperuser
   ```

   La herramienta solicita el correo (*Email*), la contraseña (*Password*) y su confirmación (*Password (again)*). Debe terminar con `Superuser created successfully.`

4. Continuar con el paso 8.

Si los registros muestran errores en lugar de avance, consultar la sección 5 (solución de problemas).

### Paso 8. Ingresar a la herramienta

1. Abrir **Firefox** en Kali e ingresar a **https://localhost:8443/**
2. Firefox mostrará la advertencia *Advertencia: riesgo potencial de seguridad a continuación*, porque la instalación local usa un certificado autofirmado. Seleccionar **Avanzado...** y luego **Aceptar el riesgo y continuar**. Esta excepción es segura únicamente porque se trata de la instalación local del estudiante.
3. Iniciar sesión con el correo y la contraseña creados en el paso 7.

### Paso 9. Uso en sesiones posteriores

Cada vez que se encienda Kali para continuar trabajando:

```bash
cd ~/ciso-assistant-community
docker compose up -d
```

Esperar uno o dos minutos e ingresar a `https://localhost:8443/`.

Para detener la herramienta al terminar la sesión:

```bash
cd ~/ciso-assistant-community
docker compose down
```

> **Importante:** `docker compose down` detiene los contenedores sin borrar la información registrada. **No** se debe usar la opción `-v` (`docker compose down -v`), porque elimina los datos de la herramienta.

### Paso 10. Comandos útiles de diagnóstico

| Comando | Para qué sirve |
|---|---|
| `docker compose ps` | Ver el estado de los contenedores. |
| `docker compose logs --tail=50` | Ver los últimos mensajes de la herramienta. |
| `docker compose restart` | Reiniciar todos los contenedores. |
| `df -h` | Ver el espacio libre en disco. |
| `free -h` | Ver la memoria disponible. |

### Paso 11. Verificación final

1. **Idioma:** en la parte inferior del menú lateral, junto al nombre del usuario, abrir el menú de tres puntos y seleccionar **Español** en la lista de idiomas.
2. **Versión:** en ese mismo menú, la opción **Acerca de CISO Assistant** muestra la versión instalada.
3. **Estado de los servicios:** en la terminal, verificar que todos los contenedores estén en ejecución y que el `backend` aparezca como `healthy`:

   ```bash
   cd ~/ciso-assistant-community
   docker compose ps
   ```

La instalación queda completa cuando se puede iniciar sesión en `https://localhost:8443/` y todos los servicios están en ejecución.

## 5. Solución de problemas

| Problema | Causa probable | Solución |
|---|---|---|
| `E: Unable to locate package docker-ce` | El repositorio de Docker no quedó registrado | Repetir el paso 4 y verificar con `cat /etc/apt/sources.list.d/docker.list` que la línea contenga `trixie`. |
| `NO_PUBKEY` o error de firma al ejecutar `sudo apt update` | La llave de Docker no se descargó correctamente | Repetir los tres comandos del numeral 1 del paso 4. |
| Conflicto de dependencias al instalar `docker-ce` | Paquetes de Docker de otra fuente instalados | Repetir el paso 3 y luego el paso 5. |
| `permission denied while trying to connect to the Docker daemon socket` | El usuario aún no pertenece al grupo `docker` en la sesión actual | Cerrar la sesión de Kali y volver a ingresar, o reiniciar la máquina virtual. Verificar con el comando `groups` que aparezca `docker`. |
| `Cannot connect to the Docker daemon` | El servicio de Docker está detenido | Ejecutar `sudo systemctl start docker` y verificar con `sudo systemctl status docker --no-pager`. |
| `docker: 'compose' is not a docker command` | Falta el complemento de Compose v2 | Ejecutar `sudo apt install -y docker-compose-plugin`. |
| `./docker-compose.sh: Permission denied` | El script no tiene permiso de ejecución | Ejecutar `chmod +x docker-compose.sh` y repetir. |
| `no space left on device` | Disco de la máquina virtual lleno | Ampliar el disco virtual a 40 GB o más, o liberar espacio con `docker system prune` (no borra los datos de contenedores en uso). |
| La herramienta está muy lenta o los contenedores se reinician | Memoria insuficiente en la máquina virtual | Apagar Kali, aumentar la memoria a 6 u 8 GB y volver a iniciar. |
| Firefox muestra "No se puede conectar" | Los contenedores aún se están iniciando o están detenidos | Esperar dos minutos y recargar; si persiste, ejecutar `docker compose ps` y `docker compose up -d`. |
| Firefox muestra la advertencia de riesgo de seguridad | Certificado autofirmado de la instalación local | Es esperado: seleccionar **Avanzado...** y **Aceptar el riesgo y continuar**. |
| El puerto 8443 está ocupado | Otro servicio de Kali usa el puerto | Identificarlo con `sudo ss -ltnp | grep 8443`, detenerlo y volver a ejecutar `docker compose up -d`. |
| Sin conexión a internet en Kali | La red de la máquina virtual no está en NAT | Cambiar el adaptador de red de la máquina virtual a NAT y reiniciar Kali. |
| `dependency failed to start: container backend is unhealthy` | El primer arranque tarda más que el tiempo de espera | Seguir el paso 7A. |
| Los registros del backend muestran `Permission denied` o `unable to open database file` | Permisos de la carpeta `db` o repositorio clonado en una carpeta compartida con el anfitrión | Verificar con `df -T .` que la carpeta esté en `ext4`; si no, clonar de nuevo dentro de `~`. Si es `ext4`: `docker compose down`, `sudo chown -R 1001:1001 db` y `docker compose up -d`. |
| Los registros muestran `database migration failed` | Base de datos inicial dañada | `docker compose down`, `docker compose rm -fs`, `sudo rm -rf db` y volver a ejecutar `./docker-compose.sh` sin `sudo`. |
| El backend se reinicia constantemente | Falta de memoria | Ejecutar `docker inspect backend --format '{{.State.OOMKilled}} {{.State.ExitCode}}'`; si muestra `true` o `137`, aumentar la memoria de la máquina virtual a 8 GB. |
| El script nunca pidió el correo ni la contraseña | El script se detuvo antes de crear el administrador | Crear el usuario con `docker compose exec backend python manage.py createsuperuser` (paso 7A, numeral 3). |
| `Error: That email is already taken` al crear el administrador | El usuario ya existe | Cambiar su contraseña con `docker compose exec backend python manage.py changepassword correo@ejemplo.com`. |
| Se olvidó la contraseña del administrador | Contraseña no registrada | Ejecutar `docker compose exec backend python manage.py changepassword correo@ejemplo.com` con el correo del administrador. |

## 6. Fuentes

- Kali Linux, "Installing Docker on Kali Linux": https://www.kali.org/docs/containers/installing-docker-on-kali/
- Docker, "Install Docker Engine on Debian": https://docs.docker.com/engine/install/debian/
- intuitem, repositorio de CISO Assistant: https://github.com/intuitem/ciso-assistant-community
- intuitem, documentación de CISO Assistant: https://intuitem.gitbook.io/ciso-assistant
