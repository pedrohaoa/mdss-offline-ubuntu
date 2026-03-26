# MDSS en Ubuntu LTS — Guía completa de despliegue offline

> **Guía paso a paso, probada en entorno real**, para desplegar OPSWAT MetaDefender Storage Security (MDSS) en Ubuntu LTS usando el **Offline Docker Toolkit (Option 2: bundled binaries)**.
>
> Dirigida a personal con poca experiencia en Linux y sin conocimiento previo de MDSS.
>
> **Última versión probada:** MDSS 4.3.1 sobre Ubuntu 24.04 LTS (noble) — marzo 2026.

---

## Tabla de contenidos

- [1. Qué es MDSS y qué vamos a hacer](#1-qué-es-mdss-y-qué-vamos-a-hacer)
- [2. Requisitos previos](#2-requisitos-previos)
- [3. Preparación del host Ubuntu](#3-preparación-del-host-ubuntu)
- [4. Estructura de directorios](#4-estructura-de-directorios)
- [5. Obtención y verificación del toolkit](#5-obtención-y-verificación-del-toolkit)
- [6. Extracción e inspección del toolkit](#6-extracción-e-inspección-del-toolkit)
- [7. Carga de imágenes Docker offline](#7-carga-de-imágenes-docker-offline)
- [8. Arranque de MDSS](#8-arranque-de-mdss)
- [9. Persistencia con systemd](#9-persistencia-con-systemd)
- [10. Validación post-reboot](#10-validación-post-reboot)
- [11. Captura de evidencias y línea base](#11-captura-de-evidencias-y-línea-base)
- [12. Troubleshooting](#12-troubleshooting)
- [13. Hardening básico para PoC](#13-hardening-básico-para-poc)
- [14. Operación diaria](#14-operación-diaria)
- [15. Upgrade path](#15-upgrade-path)
- [16. Gotchas — lo que ninguna doc oficial te dice](#16-gotchas--lo-que-ninguna-doc-oficial-te-dice)
- [17. Referencias](#17-referencias)
- [Licencia](#licencia)

---

## 1. Qué es MDSS y qué vamos a hacer

### 1.1 MetaDefender Storage Security en una frase

**MetaDefender Storage Security (MDSS)** es la solución de OPSWAT que escanea, detecta y remedia amenazas en repositorios de almacenamiento — on-premises y cloud — antes de que el malware, las vulnerabilidades o los datos sensibles expuestos se conviertan en un incidente.

### 1.2 El problema que resuelve

Las organizaciones almacenan datos en decenas de repositorios (NAS, cloud, backups, file shares) pero rara vez escanean esos datos en reposo de forma continua. Los riesgos reales:

- **Malware dormido** en ficheros legacy que nunca se re-escanearon con motores actualizados.
- **Amenazas zero-day** que las soluciones de perímetro no detectaron en su momento.
- **Datos sensibles expuestos** (PII, credenciales, secretos cloud) dentro de ficheros almacenados.
- **Backups infectados** que restaurarían el malware junto con los datos.
- **Requisitos de cumplimiento** (GDPR, NIS2, HIPAA, PCI-DSS, DORA) que exigen escaneo periódico de datos en reposo.

### 1.3 Cómo funciona — tres fases

| Fase | Qué hace | Detalle |
|------|----------|---------|
| **Integrar** | Conectar repositorios | NAS (SMB, NFS, SFTP, FTP), cloud (AWS S3, Azure Blob, Azure Files, Google Cloud, Alibaba Cloud), colaboración (OneDrive/SharePoint via Graph API, Box), S3-compatible (Wasabi, MinIO, NetApp StorageGRID). |
| **Escanear** | Analizar ficheros | Multiscanning con 30+ motores antivirus simultáneos (99.2% de detección verificada por SE Labs), Deep CDR (sanitización de ficheros), detección de vulnerabilidades basadas en ficheros, Proactive DLP (datos sensibles), Sandbox adaptativo. |
| **Remediar** | Actuar sobre resultados | Workflows automatizados: etiquetar, copiar, mover, eliminar ficheros según el veredicto. Hasta 5 destinos de remediación. Escaneo en tiempo real, programado y bajo demanda. |

### 1.4 Tecnologías core de OPSWAT integradas

| Tecnología | Función |
|-----------|---------|
| **Metascan™ Multiscanning** | 30+ motores AV simultáneos para máxima detección |
| **Deep CDR™** | Desarma y reconstruye ficheros eliminando contenido activo (macros, scripts, objetos embebidos) — previene amenazas conocidas y desconocidas |
| **Proactive DLP** | Detecta datos sensibles: SSN, tarjetas de crédito, IPs, credenciales cloud (AWS, Azure, GCP), PHI/PII en ficheros DICOM, expresiones regulares personalizadas |
| **Sandbox adaptativo** | Análisis dinámico de comportamiento para amenazas evasivas |
| **Vulnerability Assessment** | Detección de vulnerabilidades (CVEs) basadas en ficheros |

### 1.5 Top casos de uso

1. **Remediación de malware y cumplimiento** — Descubrir y eliminar amenazas dormidas en datos legacy. Escaneos en tiempo real, programados y bajo demanda con generación de informes de auditoría (HIPAA, GDPR, PCI-DSS).

2. **Seguridad de NAS y file storage** — Escaneo directo de ficheros en dispositivos NAS (NetApp, Dell EMC, Isilon) vía SMB/NFS/SFTP. Detección y remediación de malware en shares y directorios de usuario.

3. **Seguridad híbrida/multi-cloud** — Escaneo consistente de ficheros antes de que entren o salgan de entornos on-premises y cloud, asegurando que los datos en reposo están limpios.

4. **Protección proactiva de backups** — Escaneo de repositorios de backup (NFS, SMB, S3) con Identity Scanning (detección incremental), verificación multi-capa y detección rápida de cambios post-incidente.

5. **Manejo de workloads tóxicos** — Transferencia segura de ficheros complejos (ISOs, WSI, MSI, contenedores) entre zonas de clasificación baja y alta, donde las inspecciones tradicionales de contenido no son efectivas.

### 1.6 Lo que esta guía cubre

Desplegar MDSS en una VM Ubuntu LTS usando el **Offline Docker Toolkit (Option 2: bundled binaries)**, que empaqueta:

- Binarios de Docker (dockerd, docker CLI, containerd, runc, docker-compose) → no necesitas instalar Docker por tu cuenta.
- Imágenes de contenedores de todos los microservicios MDSS → no necesitas acceso a Docker Hub.
- Un script de orquestación (`mdss.sh`) → una sola herramienta para arrancar, parar, verificar y operar MDSS.

**¿Por qué este método?**

| Razón | Detalle |
|-------|---------|
| Entornos air-gapped | La VM puede no tener acceso a internet. El toolkit es autocontenido. |
| Simplicidad | Un solo ZIP, un solo directorio de trabajo, un solo script. |
| Independencia de repos | No necesitas configurar repositorios apt de Docker ni acceso a Docker Hub. |
| Consistencia | El mismo paquete produce el mismo resultado en cualquier Linux x86_64 con kernel compatible. |

---

## 2. Requisitos previos

### 2.1 Hardware mínimo y recomendado

| Recurso | Mínimo | Recomendado | Notas |
|---------|--------|-------------|-------|
| vCPU | 4 | 8-16 | MDSS arranca ~50 contenedores |
| RAM | 8 GiB | 16-32 GiB | En runtime real consume ~2-3 GiB; PostgreSQL usa ~25% de RAM para buffers |
| Disco | 50 GiB libres | 200+ GiB | El toolkit ZIP pesa ~4.4 GiB, el tar interno ~4.3 GiB, y las imágenes extraídas ocupan espacio adicional |
| Swap | Recomendado | 4 GiB | Colchón de seguridad |

> **Dato real de esta instalación:** Con 16 vCPU, 31 GiB RAM y 491 GiB de disco, el sistema queda al 7% de uso de RAM y 2% de disco tras el despliegue.

### 2.2 Sistema operativo

- **Ubuntu 24.04 LTS** (22.04 LTS también debería funcionar — no probado en esta guía).
- Arquitectura **x86_64**.
- Kernel 5.x o superior (Ubuntu 24.04 trae 6.8.x).

### 2.3 Artefactos a obtener antes de empezar

Descárgalos desde el portal [MyOPSWAT](https://my.opswat.com/) (requiere cuenta):

| Artefacto | Dónde encontrarlo |
|-----------|--------------------|
| `offline-docker-toolkit-<VERSION>.zip` | MyOPSWAT → Downloads → MetaDefender Storage Security → Offline Docker Toolkit |
| SHA256 del ZIP | Publicado en la página de descarga o en las release notes |
| Clave de licencia MDSS | MyOPSWAT → Licenses |

### 2.4 Acceso al host

- Usuario con permisos `sudo` (grupo `sudo` en Ubuntu).
- Acceso SSH o consola directa.

---

## 3. Preparación del host Ubuntu

### 3.1 Verificar el sistema

```bash
echo "=== Verificación del sistema ==="
echo "CPUs: $(nproc)"
free -h
df -h /
swapon --show
uname -r
lsb_release -a
```

Confirmar que hay suficiente RAM, disco y CPUs según la tabla de requisitos.

### 3.2 Paquetes prerrequisito

Verificar qué paquetes están instalados y cuáles faltan:

```bash
for pkg in curl openssl sudo ca-certificates jq gnupg2 iproute2 unzip wget iptables net-tools; do
  dpkg -s "$pkg" &>/dev/null && echo "[OK] $pkg" || echo "[FALTA] $pkg"
done
```

Instalar los que falten:

```bash
# Si la VM tiene internet:
sudo apt update
sudo apt install -y jq gnupg2 unzip wget net-tools

# Si la VM NO tiene internet:
# Descargar los .deb en otra máquina y transferirlos via SCP/USB.
# Instalar con: sudo dpkg -i <paquete>.deb
```

> **En Ubuntu 24.04 Server**, los que suelen faltar son `jq`, `unzip`, `gnupg2` y `net-tools`. El resto viene preinstalado.

### 3.3 Hostname y /etc/hosts

```bash
# Verificar hostname actual
hostnamectl

# Añadir resolución local (sustituir IP y hostname por los tuyos)
grep $(hostname) /etc/hosts || echo "$(hostname -I | awk '{print $1}')  $(hostname)" | sudo tee -a /etc/hosts
```

### 3.4 NTP

```bash
timedatectl
```

Confirmar que `System clock synchronized: yes` y `NTP service: active`. Ubuntu 24.04 viene con `systemd-timesyncd` activo por defecto. Si no está sincronizado:

```bash
sudo timedatectl set-ntp true
```

### 3.5 Ajustar ulimits (importante)

Ubuntu 24.04 trae `ulimit -n` en **1024** por defecto, que es insuficiente para Docker con muchos contenedores.

```bash
# Verificar valor actual
ulimit -n
```

Si devuelve 1024 (o menos de 65536):

```bash
sudo tee /etc/security/limits.d/99-mdss.conf > /dev/null <<'EOF'
*    soft    nofile    65536
*    hard    nofile    65536
root soft    nofile    65536
root hard    nofile    65536
EOF
```

> ⚠️ **Este cambio requiere cerrar sesión y volver a entrar** (logout/login SSH completo). No basta con `source` ni `su -`.

Tras reconectar:

```bash
ulimit -n
# Expected: 65536
```

### 3.6 Ajustar parámetros de kernel (sysctl)

```bash
sudo tee /etc/sysctl.d/99-mdss.conf > /dev/null <<'EOF'
# MDSS / Docker optimizations
net.core.somaxconn = 1024
net.ipv4.ip_forward = 1
vm.max_map_count = 262144
fs.file-max = 262144
EOF

sudo sysctl --system
```

Verificar:

```bash
sysctl net.ipv4.ip_forward vm.max_map_count fs.file-max net.core.somaxconn
```

Resultado esperado:

```
net.ipv4.ip_forward = 1
vm.max_map_count = 262144
fs.file-max = 262144
net.core.somaxconn = 1024
```

### 3.7 Crear directorio de logs de MDSS

El script `mdss.sh` intenta escribir logs en `/var/log/mdss/` pero **no crea el directorio automáticamente**. Si no existe, verás un error no bloqueante en la primera ejecución.

```bash
sudo mkdir -p /var/log/mdss
sudo chmod 0755 /var/log/mdss
```

---

## 4. Estructura de directorios

Todo el despliegue vive bajo `/srv/mdss-offline/`. Esta guía asume una sola partición montada en `/` — adaptar si hay particiones dedicadas.

```
/srv/mdss-offline/                  ← Raíz del despliegue
├── stage/                          ← ZIPs descargados + checksums
├── package/
│   └── current/                    ← Toolkit extraído (PATH OPERATIVO FIJO)
│       ├── mdss.sh                 ← Script de orquestación
│       ├── mdss.tar                ← Imágenes Docker empaquetadas (~4.3 GiB)
│       ├── .env                    ← Variables de configuración
│       ├── customer.env            ← Overrides del cliente (auto-generado)
│       ├── docker/                 ← Binarios Docker bundled
│       ├── docker-compose.*.yml    ← Definiciones de servicios
│       ├── configurations/         ← Config de la aplicación
│       ├── rabbitmq/               ← Config de RabbitMQ
│       └── webclient/              ← Config de nginx
└── backup/                         ← Backups y evidencias
```

Crear la estructura:

```bash
sudo mkdir -p /srv/mdss-offline/{stage,package/current,backup}
sudo chown -R root:root /srv/mdss-offline
sudo chmod -R 0755 /srv/mdss-offline
```

Verificar:

```bash
find /srv/mdss-offline -maxdepth 3 -type d | sort
```

> **Importante:** Una vez extraído el toolkit en `package/current/`, **no mover ni renombrar ese directorio**. El toolkit resuelve rutas relativas a su ubicación.

---

## 5. Obtención y verificación del toolkit

### 5.1 Transferir el ZIP al host

```bash
# Variables de sesión (ajustar versión según tu descarga)
export MDSS_VERSION="4.3.1"
export MDSS_TOOLKIT_ZIP="offline-docker-toolkit-${MDSS_VERSION}.zip"
```

**Si la VM tiene internet** — descargar directamente en stage:

```bash
cd /srv/mdss-offline/stage
# sudo wget -O "${MDSS_TOOLKIT_ZIP}" "<URL_de_MyOPSWAT>"
```

**Si la VM NO tiene internet** — transferir desde otra máquina:

```bash
# Desde la máquina con el ZIP:
scp offline-docker-toolkit-4.3.1.zip usuario@<IP_VM>:/tmp/

# En la VM:
sudo mv /tmp/offline-docker-toolkit-4.3.1.zip /srv/mdss-offline/stage/
```

### 5.2 Verificar integridad SHA256

```bash
cd /srv/mdss-offline/stage
sha256sum "${MDSS_TOOLKIT_ZIP}"
```

Comparar el hash con el publicado en MyOPSWAT. Para automatizar:

```bash
# Sustituir <HASH_OFICIAL> por el valor real (case-insensitive)
echo "<HASH_OFICIAL>  ${MDSS_TOOLKIT_ZIP}" | sha256sum -c -
```

Resultado esperado:

```
offline-docker-toolkit-4.3.1.zip: OK
```

> ⚠️ **Si el checksum falla: NO continuar.** Re-transferir el fichero. Si persiste, re-descargar desde MyOPSWAT. Un ZIP corrupto producirá imágenes Docker inválidas.

---

## 6. Extracción e inspección del toolkit

### 6.1 Extraer

```bash
export MDSS_HOME="/srv/mdss-offline/package/current"

sudo rm -rf "${MDSS_HOME:?}"/*
cd "${MDSS_HOME}"
sudo unzip "/srv/mdss-offline/stage/${MDSS_TOOLKIT_ZIP}"
```

### 6.2 Dar permisos de ejecución

```bash
sudo chmod +x ./mdss.sh
sudo chmod +x ./docker/*
```

### 6.3 Inspeccionar el contenido

```bash
echo "=== Script principal ==="
ls -lh mdss.sh
file mdss.sh

echo "=== Imágenes empaquetadas ==="
ls -lh mdss.tar

echo "=== Binarios Docker bundled ==="
ls -lh docker/

echo "=== Docker version ==="
./docker/docker --version
```

Resultado esperado:

- `mdss.sh` → Bash script, ~108 KB, ejecutable.
- `mdss.tar` → ~4.3 GiB (contiene todas las imágenes Docker).
- `docker/` → 9 binarios (~255 MB total): `docker`, `dockerd`, `containerd`, `containerd-shim-runc-v2`, `ctr`, `docker-compose`, `docker-init`, `docker-proxy`, `runc`.
- Docker version → 27.3.1 (puede variar con el toolkit).

### 6.4 Verificar que no hay Docker de sistema

```bash
which docker 2>/dev/null && echo "⚠️  Docker de sistema detectado — posible conflicto" || echo "✅ Sin Docker de sistema"
systemctl is-active docker 2>/dev/null && echo "⚠️  docker.service activo" || echo "✅ Sin docker.service activo"
```

Si hay Docker de sistema instalado, desactivarlo:

```bash
sudo systemctl stop docker docker.socket
sudo systemctl disable docker docker.socket
```

---

## 7. Carga de imágenes Docker offline

```bash
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -u load_images
```

Este proceso carga los ~4.3 GiB de imágenes desde `mdss.tar` en el Docker daemon local. **Puede tardar 3-7 minutos** dependiendo de la velocidad del disco.

Resultado esperado:

```
[announce] Loading docker images...
[announce] Please wait, downloading, this can take a few minutes...
[announce] Finished loading images.
```

Verificar las imágenes cargadas:

```bash
sudo /srv/mdss-offline/package/current/docker/docker images
```

> ⚠️ **Gotcha:** No usar `sudo ./docker/docker` (ruta relativa). Usar siempre la **ruta absoluta** con sudo: `sudo /srv/mdss-offline/package/current/docker/docker`. Ver sección [Gotchas](#16-gotchas--lo-que-ninguna-doc-oficial-te-dice) para la explicación.

Resultado esperado: ~57 imágenes con tags `4.3.1` (microservicios MDSS) + imágenes de infraestructura (postgres, redis, rabbitmq, mongo).

---

## 8. Arranque de MDSS

```bash
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c start
```

**Puede tardar 1-3 minutos.** El script:

1. Verifica inotify limits.
2. Detecta que HTTPS no está activo (normal en primera instalación).
3. Auto-configura PostgreSQL (shared_buffers al ~25% de RAM, effective_cache_size al ~50%).
4. Genera passwords aleatorias para PostgreSQL en `customer.env`.
5. Levanta todos los contenedores con docker-compose.

Resultado esperado:

```
[announce] Setting POSTGRES_SHARED_BUFFERS to <valor>MB
[announce] Setting POSTGRES_EFFECTIVE_CACHE_SIZE to <valor>MB
[announce] Running docker compose up for all enabled components...
[announce] The web interface is available at http://mds_host
[announce] MetaDefender Storage Security has started
```

### 8.1 Verificación completa

```bash
# Estado de MDSS
sudo ./mdss.sh -c status
# Expected: "MetaDefender Storage Security is running."

# Contenedores
sudo /srv/mdss-offline/package/current/docker/docker ps --format "table {{.Names}}\t{{.Status}}" | head -30
# Expected: todos los contenedores en "Up"

# Puertos
sudo ss -ltnp | grep -E ':(80|443)\s'
# Expected: puerto 80 y 443 en LISTEN

# Acceso web local
curl -sI http://127.0.0.1/ | head -5
# Expected: HTTP/1.1 200 OK
```

### 8.2 Acceso web

Abrir en un navegador: `http://<IP_de_tu_VM>/`

En la primera conexión, MDSS te pedirá crear el usuario administrador.

---

## 9. Persistencia con systemd

Sin un servicio systemd, MDSS no arrancará automáticamente tras un reboot.

### 9.1 Crear la unidad

```bash
sudo tee /etc/systemd/system/mdss-offline.service > /dev/null <<'EOF'
[Unit]
Description=OPSWAT MetaDefender Storage Security (Offline Docker Toolkit)
Documentation=https://www.opswat.com/docs/mdss
After=network-online.target
Wants=network-online.target
ConditionPathExists=/srv/mdss-offline/package/current/mdss.sh

[Service]
Type=oneshot
RemainAfterExit=yes
User=root
Group=root
WorkingDirectory=/srv/mdss-offline/package/current

ExecStart=/usr/bin/bash /srv/mdss-offline/package/current/mdss.sh -c start
ExecStop=/usr/bin/bash /srv/mdss-offline/package/current/mdss.sh -c stop
ExecReload=/usr/bin/bash /srv/mdss-offline/package/current/mdss.sh -c restart

TimeoutStartSec=1800
TimeoutStopSec=1800

StandardOutput=journal
StandardError=journal
SyslogIdentifier=mdss-offline

[Install]
WantedBy=multi-user.target
EOF
```

### 9.2 Habilitar

```bash
sudo systemctl daemon-reload
sudo systemctl enable mdss-offline.service
systemctl is-enabled mdss-offline.service
# Expected: enabled
```

> **Nota:** Si MDSS ya está corriendo (lo arrancaste en el paso 8), no necesitas hacer `systemctl start` ahora. El servicio arrancará automáticamente en el próximo boot.

---

## 10. Validación post-reboot

Este paso es **crítico** para una PoC. Demuestra que el despliegue sobrevive a un reinicio sin intervención manual.

### 10.1 Reboot controlado

```bash
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c stop
sync
sudo systemctl reboot
```

### 10.2 Verificación tras reconectar

```bash
# Servicio systemd
sudo systemctl status mdss-offline.service
# Expected: active (exited), status=0/SUCCESS

# MDSS status
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c status
# Expected: "MetaDefender Storage Security is running."

# Contenedores
sudo /srv/mdss-offline/package/current/docker/docker ps | head -10
# Expected: contenedores en "Up"

# Acceso web
curl -sI http://127.0.0.1/ | head -3
# Expected: HTTP/1.1 200 OK
```

---

## 11. Captura de evidencias y línea base

Guardar una snapshot del estado post-instalación. Útil para comparar ante problemas futuros o para documentar la PoC.

> ⚠️ **Gotcha:** Usar `| sudo tee` en lugar de `>` para las redirecciones. El directorio es propiedad de root, y `>` la ejecuta tu shell sin privilegios. Ver sección [Gotchas](#16-gotchas--lo-que-ninguna-doc-oficial-te-dice).

```bash
export EVIDENCE_DIR="/srv/mdss-offline/backup/evidence-$(date +%Y%m%d-%H%M%S)"
sudo mkdir -p "${EVIDENCE_DIR}"

# Sistema
hostnamectl | sudo tee "${EVIDENCE_DIR}/hostnamectl.txt" > /dev/null
uname -a | sudo tee "${EVIDENCE_DIR}/uname.txt" > /dev/null
free -h | sudo tee "${EVIDENCE_DIR}/memory.txt" > /dev/null
df -h | sudo tee "${EVIDENCE_DIR}/disk.txt" > /dev/null
ip -4 addr show | sudo tee "${EVIDENCE_DIR}/network.txt" > /dev/null
sudo ss -ltnp | sudo tee "${EVIDENCE_DIR}/ports.txt" > /dev/null

# MDSS
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c status 2>&1 | sudo tee "${EVIDENCE_DIR}/mdss-status.txt" > /dev/null
sudo /srv/mdss-offline/package/current/docker/docker ps -a 2>&1 | sudo tee "${EVIDENCE_DIR}/docker-ps.txt" > /dev/null
sudo /srv/mdss-offline/package/current/docker/docker images 2>&1 | sudo tee "${EVIDENCE_DIR}/docker-images.txt" > /dev/null
sha256sum /srv/mdss-offline/stage/*.zip | sudo tee "${EVIDENCE_DIR}/toolkit-sha256.txt" > /dev/null
systemctl status mdss-offline.service 2>&1 | sudo tee "${EVIDENCE_DIR}/systemd-status.txt" > /dev/null

# Verificar
ls -la "${EVIDENCE_DIR}/"
```

Resultado esperado: 11 ficheros, todos con tamaño > 0 bytes.

---

## 12. Troubleshooting

### 12.1 `mdss.sh -u load_images` falla

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `No such file or directory` | Nombre del tar difiere | `ls *.tar` y usar el nombre real |
| `invalid tar header` | ZIP corrupto o transferencia incompleta | Re-verificar SHA256, re-transferir |
| `Error processing tar file` | Disco lleno | `df -h /` — liberar espacio (necesitas ~9 GiB temporales: ZIP + tar) |
| `permission denied` | Falta sudo | Ejecutar con `sudo` |

Diagnóstico:

```bash
cd /srv/mdss-offline/package/current
ls -lh mdss.tar
file mdss.tar
df -h /
```

### 12.2 `mdss.sh -c start` falla

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `Permission denied` | Falta `chmod +x` | `sudo chmod +x mdss.sh` |
| `bad interpreter` | Fin de línea Windows | `sudo sed -i 's/\r$//' mdss.sh` |
| `docker: command not found` | Binarios sin permisos | `sudo chmod +x docker/*` |

Diagnóstico en modo debug:

```bash
sudo bash -x ./mdss.sh -c start 2>&1 | head -100
```

### 12.3 Contenedores en Restarting o Exited

```bash
# Ver todos los contenedores
sudo /srv/mdss-offline/package/current/docker/docker ps -a

# Logs de un contenedor que falla
sudo /srv/mdss-offline/package/current/docker/docker logs <NOMBRE_CONTENEDOR> --tail 100
```

| Contenedor | Causa típica | Solución |
|-----------|-------------|----------|
| PostgreSQL | Permisos en data dir | `sudo chown -R 999:999 <path>` |
| RabbitMQ | Hostname cambió | Limpiar data de RabbitMQ y reiniciar |
| Backend/API | DB no lista | Verificar que PostgreSQL está `Up` primero |
| Cualquiera | OOM killed | `dmesg \| grep -i oom` — verificar RAM |

### 12.4 UI no accesible desde el navegador

```bash
# 1. ¿Responde localmente?
curl -sI http://127.0.0.1/ | head -5

# 2. ¿Puertos escuchando?
sudo ss -ltnp | grep -E ':(80|443)\s'

# 3. ¿Firewall bloqueando?
sudo ufw status
```

| Resultado | Causa | Solución |
|-----------|-------|----------|
| OK local, falla remoto | Firewall | `sudo ufw allow 80/tcp && sudo ufw allow 443/tcp` |
| Connection refused local | Contenedor web caído | `sudo /srv/.../docker/docker logs mdss_webclient --tail 50` |
| Timeout remoto | Security group / red | Verificar reglas de red de la VM (AWS, Azure, etc.) |

### 12.5 Error de logs: `tee: /var/log/mdss/...: No such file or directory`

No es bloqueante, pero indica que no se creó el directorio de logs:

```bash
sudo mkdir -p /var/log/mdss
sudo chmod 0755 /var/log/mdss
```

### 12.6 Recopilar información para soporte OPSWAT

```bash
SUPPORT_DIR="/tmp/mdss-support-$(date +%Y%m%d-%H%M%S)"
mkdir -p "${SUPPORT_DIR}"

cd /srv/mdss-offline/package/current
uname -a > "${SUPPORT_DIR}/system.txt"
free -h >> "${SUPPORT_DIR}/system.txt"
df -h >> "${SUPPORT_DIR}/system.txt"

sudo ./mdss.sh -c status > "${SUPPORT_DIR}/mdss-status.txt" 2>&1
sudo /srv/mdss-offline/package/current/docker/docker ps -a > "${SUPPORT_DIR}/docker-ps.txt" 2>&1

for c in $(sudo /srv/mdss-offline/package/current/docker/docker ps -a --format '{{.Names}}'); do
  sudo /srv/mdss-offline/package/current/docker/docker logs "$c" --tail 500 > "${SUPPORT_DIR}/logs-${c}.txt" 2>&1
done

sudo journalctl -u mdss-offline --no-pager -n 500 > "${SUPPORT_DIR}/journal.txt" 2>&1

tar czf "${SUPPORT_DIR}.tar.gz" -C /tmp "$(basename ${SUPPORT_DIR})"
echo "Paquete de soporte: ${SUPPORT_DIR}.tar.gz ($(du -sh ${SUPPORT_DIR}.tar.gz | cut -f1))"
```

---

## 13. Hardening básico para PoC

### 13.1 Firewall (ufw)

```bash
# Verificar estado
sudo ufw status

# Si está inactivo y quieres activarlo:
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp comment "MDSS HTTP"
sudo ufw allow 443/tcp comment "MDSS HTTPS"
sudo ufw enable
sudo ufw status numbered
```

> ⚠️ **Docker y ufw:** Docker modifica iptables directamente y puede bypasear reglas ufw. Para una PoC es aceptable. Para producción, investigar el parche [ufw-docker](https://github.com/chaifeng/ufw-docker).

### 13.2 SSH

```bash
# Deshabilitar login root por SSH
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload ssh
```

### 13.3 Logging — retención de journald

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/mdss.conf > /dev/null <<'EOF'
[Journal]
SystemMaxUse=2G
SystemKeepFree=1G
MaxRetentionSec=30day
Compress=yes
EOF

sudo systemctl restart systemd-journald
```

### 13.4 Integridad de binarios post-instalación

Guardar fingerprints para detectar manipulaciones futuras:

```bash
cd /srv/mdss-offline/package/current
sha256sum mdss.sh docker/* | sudo tee /srv/mdss-offline/backup/binaries-sha256.txt > /dev/null
```

Para verificar más adelante:

```bash
cd /srv/mdss-offline/package/current
sha256sum -c /srv/mdss-offline/backup/binaries-sha256.txt
```

### 13.5 Backup de configuración

```bash
sudo tar czf /srv/mdss-offline/backup/config-$(date +%Y%m%d).tar.gz \
  /etc/systemd/system/mdss-offline.service \
  /etc/sysctl.d/99-mdss.conf \
  /etc/security/limits.d/99-mdss.conf \
  /srv/mdss-offline/package/current/.env \
  /srv/mdss-offline/package/current/customer.env \
  2>/dev/null
```

---

## 14. Operación diaria

### 14.1 Path operativo fijo

**Siempre** operar desde este directorio:

```bash
cd /srv/mdss-offline/package/current
```

### 14.2 Comandos esenciales

| Acción | Comando |
|--------|---------|
| Ver estado | `sudo ./mdss.sh -c status` |
| Arrancar | `sudo ./mdss.sh -c start` |
| Parar | `sudo ./mdss.sh -c stop` |
| Reiniciar | `sudo ./mdss.sh -c restart` |
| Ver contenedores | `sudo /srv/mdss-offline/package/current/docker/docker ps` |
| Ver logs de un contenedor | `sudo /srv/mdss-offline/package/current/docker/docker logs <NOMBRE> --tail 100` |
| Ver imágenes | `sudo /srv/mdss-offline/package/current/docker/docker images` |

### 14.3 Apagado limpio

**Siempre parar MDSS antes de apagar o reiniciar la VM.** El stack incluye PostgreSQL, RabbitMQ y Redis, que pueden corromperse con un apagado brusco.

```bash
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c stop
sync
sudo systemctl poweroff   # o reboot
```

### 14.4 Monitorización rápida

```bash
echo "=== MDSS Health Check — $(date) ==="
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c status
echo "--- Contenedores con problemas ---"
sudo /srv/mdss-offline/package/current/docker/docker ps -a --filter "status=exited" --filter "status=restarting" --format "{{.Names}}: {{.Status}}"
echo "--- Recursos ---"
free -h | grep Mem
df -h / | tail -1
echo "--- Puertos ---"
sudo ss -ltnp | grep -E ':(80|443)\s'
```

---

## 15. Upgrade path

### 15.1 Preparación

1. Descargar el nuevo `offline-docker-toolkit-<NUEVA_VERSION>.zip` desde MyOPSWAT.
2. Verificar SHA256.
3. Transferir al host y colocar en `/srv/mdss-offline/stage/`.

### 15.2 Procedimiento

```bash
# Variables
export NEW_VERSION="4.4.0"  # Sustituir por la versión real
export NEW_ZIP="offline-docker-toolkit-${NEW_VERSION}.zip"

# 1. Backup de la versión actual
sudo cp -a /srv/mdss-offline/package/current /srv/mdss-offline/backup/package-$(date +%Y%m%d)

# 2. Backup de customer.env (contiene passwords de PostgreSQL auto-generadas)
sudo cp /srv/mdss-offline/package/current/customer.env /srv/mdss-offline/backup/customer.env.$(date +%Y%m%d)

# 3. Parar MDSS
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c stop

# 4. Extraer nueva versión
sudo rm -rf /srv/mdss-offline/package/current/*
cd /srv/mdss-offline/package/current
sudo unzip "/srv/mdss-offline/stage/${NEW_ZIP}"
sudo chmod +x ./mdss.sh ./docker/*

# 5. Restaurar customer.env (para mantener passwords de DB)
sudo cp /srv/mdss-offline/backup/customer.env.$(date +%Y%m%d) /srv/mdss-offline/package/current/customer.env

# 6. Cargar nuevas imágenes
sudo ./mdss.sh -u load_images

# 7. Arrancar
sudo ./mdss.sh -c start

# 8. Verificar
sudo ./mdss.sh -c status
sudo /srv/mdss-offline/package/current/docker/docker ps | head -10
curl -sI http://127.0.0.1/ | head -3
```

> ⚠️ **Siempre hacer backup de `customer.env` antes de un upgrade.** Contiene las passwords de PostgreSQL generadas en la primera instalación. Sin ese fichero, la nueva versión no puede conectar a la base de datos existente.

### 15.3 Rollback

Si el upgrade falla:

```bash
cd /srv/mdss-offline/package/current
sudo ./mdss.sh -c stop

# Restaurar versión anterior
sudo rm -rf /srv/mdss-offline/package/current/*
sudo cp -a /srv/mdss-offline/backup/package-<FECHA>/* /srv/mdss-offline/package/current/

# Cargar imágenes antiguas
sudo ./mdss.sh -u load_images
sudo ./mdss.sh -c start
```

---

## 16. Gotchas — lo que ninguna doc oficial te dice

Lecciones aprendidas durante un despliegue real. Si vienes de RHEL o si es tu primer despliegue de MDSS, lee esta sección.

### G1. `sudo ./docker/docker` → command not found

**Problema:** `sudo` con ruta relativa (`./docker/docker`) falla porque `sudo` busca el binario a través de `secure_path`, no en el directorio actual.

**Solución:** Usar siempre la ruta absoluta:

```bash
sudo /srv/mdss-offline/package/current/docker/docker images
```

O crear un alias de sesión:

```bash
alias mdocker="sudo /srv/mdss-offline/package/current/docker/docker"
mdocker ps
```

### G2. `/var/log/mdss/` no existe → error en primera ejecución

**Problema:** `mdss.sh` intenta escribir logs con `tee` en `/var/log/mdss/<timestamp>.log` pero no crea el directorio. Produce: `tee: /var/log/mdss/...: No such file or directory`.

**Impacto:** No bloqueante — MDSS arranca igual, pero pierdes los logs del script.

**Solución:** Crear el directorio antes de la primera ejecución de `mdss.sh` (ver sección 3.7).

### G3. `ulimit -n 1024` → no aplica tras editar limits.d

**Problema:** Editaste `/etc/security/limits.d/99-mdss.conf` pero `ulimit -n` sigue en 1024.

**Causa:** Los ficheros en `limits.d` los procesa PAM al inicio de sesión. Un `source`, `su -` o `bash` no los re-aplica.

**Solución:** Cerrar sesión SSH completamente y reconectar.

### G4. Redirección `>` a directorio root → Permission denied

**Problema:** `sudo mkdir -p /dir && hostnamectl > /dir/file.txt` falla porque `>` la ejecuta tu shell (sin privilegios), no sudo.

**Solución:** Usar `| sudo tee`:

```bash
hostnamectl | sudo tee /dir/file.txt > /dev/null
```

### G5. HTTPS desactivado por defecto

**Problema:** El primer arranque muestra `HTTPS is not active. Updating configuration.` MDSS arranca en HTTP.

**Impacto:** Para una PoC interna es aceptable. Para exposición a usuarios, configurar TLS.

**Solución:** Configurar certificados en la UI de MDSS o mediante reverse proxy (nginx/Apache con TLS terminado fuera de MDSS).

### G6. PostgreSQL auto-tuning: no tocar (en PoC)

El script `mdss.sh` calcula automáticamente `shared_buffers` (~25% de RAM) y `effective_cache_size` (~50% de RAM) y los escribe en `customer.env`. Estos valores son óptimos para la mayoría de escenarios. No ajustarlos manualmente a menos que haya una razón técnica documentada.

### G7. 57 imágenes Docker + 9 GiB de espacio temporal

El ZIP pesa ~4.4 GiB y el tar interno ~4.3 GiB. Se necesitan ~9 GiB libres solo en la fase de extracción (stage + current). Planificar el espacio en disco considerando este overhead.

### G8. Si vienes de RHEL — qué NO hacer en Ubuntu

| RHEL | Ubuntu | Qué hacer |
|------|--------|-----------|
| `dnf install` / `yum install` | `apt install` | Cambiar el package manager |
| `semanage`, `restorecon`, `chcon` | No existe | Eliminar — Ubuntu usa AppArmor, no SELinux |
| `firewall-cmd` | `ufw` | Reescribir — no hay traducción 1:1 |
| `grep /var/log/messages` | `journalctl` o `/var/log/syslog` | Cambiar la fuente de logs |
| `systemctl enable --now` | `systemctl enable` + `systemctl start` | Funciona igual, pero separarlos mejora la depuración |

---

## 17. Referencias

- [Documentación oficial MDSS — Instalación offline Linux](https://www.opswat.com/docs/mdss/installation/linux-offline-installation)
- [Documentación oficial MDSS — Requisitos del sistema](https://www.opswat.com/docs/mdss/getting-started/general-system-requirements)
- [Portal MyOPSWAT — Descargas y licencias](https://my.opswat.com/)
- [Docker — Instalación en Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [ufw-docker — Solución al bypass de firewall](https://github.com/chaifeng/ufw-docker)

---

## Licencia

Este documento se distribuye bajo [MIT License](LICENSE).

No tiene afiliación oficial con OPSWAT. MetaDefender y MDSS son marcas registradas de OPSWAT, Inc.

---

> **Mantenido por:** [@pedrohaoa](https://github.com/pedrohaoa)
>
> **Repo:** [mdss-offline-ubuntu](https://github.com/pedrohaoa/mdss-offline-ubuntu)
>
> **Última versión probada:** Ubuntu 24.04 LTS, MDSS 4.3.1, Offline Docker Toolkit, marzo 2026.
>
> **Contribuciones bienvenidas** — abre un issue o un PR si encuentras errores o tienes mejoras.
