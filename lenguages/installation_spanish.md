# Manual de Instalación: Servidor de Videovigilancia (MotionEye)

Este documento detalla el despliegue del sistema de videovigilancia local utilizando Docker y MotionEye sobre Ubuntu Server. El sistema está diseñado para ingerir video por protocolo RTSP (ej. cámaras Tapo C120), procesar la detección de movimiento en memoria RAM para cuidar los IOPS del disco, y mantener una rotación de archivos automática de 7 días.

---

## 1. Preparación del Entorno

Asegúrate de tener el disco duro de 1TB (conectado por SATA a la placa) debidamente montado en el sistema operativo. Para este manual, asumiremos que el punto de montaje destinado a los videos es `/media/seguridad/videos`.

### 1.1. Creación de directorios
Crea la estructura de carpetas necesaria para la configuración y el almacenamiento:

```bash
# Directorio para la configuración de MotionEye
sudo mkdir -p /opt/motioneye/conf

# Directorio principal para el almacenamiento de los videos (En tu disco de 1TB)
sudo mkdir -p /media/seguridad/videos

# Directorio para los scripts de mantenimiento
sudo mkdir -p /opt/scripts/seguridad

```

---

## 2. Despliegue del Contenedor (Docker)

Se utilizará un volumen temporal en RAM (`tmpfs`) para que MotionEye procese los frames de video en vivo sin saturar la aguja de lectura/escritura del disco duro mecánico.

### 2.1. Archivo `docker-compose.yml`

Crea el archivo de orquestación en la ruta de tu preferencia (ej. `/opt/motioneye/docker-compose.yml`):

```yaml
version: '3'
services:
  motioneye:
    image: ccrisan/motioneye:master-amd64
    container_name: motioneye
    restart: unless-stopped
    ports:
      - "8765:8765"
    volumes:
      # Sincronización de zona horaria con el host
      - /etc/localtime:/etc/localtime:ro
      # Persistencia de configuración
      - /opt/motioneye/conf:/etc/motioneye
      # Persistencia de videos (Apunta al disco duro de 1TB)
      - /media/seguridad/videos:/var/lib/motioneye
    tmpfs:
      # Procesamiento de video en vivo en memoria RAM para cuidar el HDD
      - /tmp:exec,mode=777

```

### 2.2. Levantar el servicio

Ejecuta el siguiente comando en el mismo directorio donde guardaste el archivo `docker-compose.yml`:

```bash
sudo docker-compose up -d

```

El servidor estará accesible a través de: `http://<IP_DEL_SERVIDOR>:8765`

---

## 3. Automatización de Retención (Limpieza de 7 días)

Para evitar que el disco se llene, el sistema ejecutará un script atómico que depura archivos antiguos y limpia la estructura de directorios generada por MotionEye.

### 3.1. Script de Limpieza

Crea el archivo `/opt/scripts/seguridad/limpieza_rotativa.sh`:

```bash
sudo nano /opt/scripts/seguridad/limpieza_rotativa.sh

```

Pega el siguiente código:

```bash
#!/bin/bash

# Ruta al almacenamiento en el disco duro
DIRECTORIO_VIDEOS="/media/seguridad/videos"

# 1. Elimina archivos (.mp4, .jpg, .thumb) con más de 7 días de antigüedad
find "$DIRECTORIO_VIDEOS" -type f \( -name "*.mp4" -o -name "*.jpg" -o -name "*.thumb" \) -mtime +7 -print -delete

# 2. Elimina directorios vacíos (carpetas de fechas antiguas)
find "$DIRECTORIO_VIDEOS" -type d -empty -delete

```

### 3.2. Permisos de ejecución

Otorga los permisos necesarios al script:

```bash
sudo chmod +x /opt/scripts/seguridad/limpieza_rotativa.sh

```

### 3.3. Programación en Cron

Añade la tarea al planificador del sistema operativo para que se ejecute todos los días en la madrugada (3:00 AM).

Abre el crontab del usuario root:

```bash
sudo crontab -e

```

Añade la siguiente línea al final del archivo:

```text
0 3 * * * /bin/bash /opt/scripts/seguridad/limpieza_rotativa.sh > /opt/scripts/seguridad/limpieza.log 2>&1

```

---

## 4. Configuración en la Interfaz Web (MotionEye)

Una vez levantado el contenedor, ingresa a la interfaz web con el usuario `admin` (sin contraseña por defecto) y realiza los siguientes ajustes para cada cámara:

1. **Añadir Cámara:**
* Tipo: `Network Camera`.
* URL (Ejemplo RTSP Tapo): `rtsp://usuario:clave@<IP_DE_LA_CAMARA>:554/stream1`.


2. **Video Resolution & Framerate:**
* Ajustar según necesidad, recomendación: 10 a 15 FPS para reducir el consumo de CPU.


3. **File Storage:**
* Asegurar que el `Root Directory` esté apuntando a `/var/lib/motioneye` (el cual está mapeado internamente a tu disco de 1TB).


4. **Movies:**
* Habilitar opción.
* `Movie Format`: **MP4** o **H.264/MP4**.
* `Recording Mode`: **Motion Triggered**.


5. **Motion Detection:**
* Habilitar opción.
* Ajustar `Frame Change Threshold`: Recomendado entre **3.5% y 5.0%** (calibrar según el movimiento deseado en el entorno físico).