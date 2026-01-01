---
layout: post
title: "Guía Completa: Análisis del Protocolo LPD con Docker y Wireshark"
date: 2026-01-01
categories: docker, Debian, wireshark, LPD 
---

## Índice
1. [Requisitos previos e instalación](#requisitos-previos)
2. [Estructura del proyecto](#estructura-del-proyecto)
3. [Configuración del entorno](#configuración-del-entorno)
4. [Puesta en marcha](#puesta-en-marcha)
5. [Captura y análisis del tráfico LPD](#captura-y-análisis)
6. [Interpretación de resultados](#interpretación-de-resultados)
7. [Script de automatización](#script-de-automatización)

---

## 1. Requisitos Previos e Instalación {#requisitos-previos}

### Instalación de Docker

#### En Debian/Ubuntu/Kali Linux:
```bash
# Actualizar repositorios
sudo apt update

# Instalar dependencias necesarias
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Añadir la clave GPG oficial de Docker
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Añadir el repositorio de Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Añadir tu usuario al grupo docker (para no usar sudo)
sudo usermod -aG docker $USER

# Aplicar cambios (o reiniciar sesión)
newgrp docker

# Verificar instalación
docker --version
```

### Instalación de Docker Compose

```bash
# Docker Compose v2 ya viene integrado con Docker Desktop
# Verificar instalación
docker compose version

# Si no está instalado (solo en sistemas antiguos):
sudo apt install -y docker-compose-plugin
```

### Herramientas adicionales necesarias

```bash
# Instalar Wireshark para análisis gráfico
sudo apt install -y wireshark

# Permitir captura sin privilegios de root (opcional)
sudo dpkg-reconfigure wireshark-common
# Seleccionar "Yes" cuando pregunte si usuarios no-root pueden capturar paquetes
sudo usermod -aG wireshark $USER
```

---

## 2. Estructura del Proyecto {#estructura-del-proyecto}

### Crear el directorio del proyecto

```bash
# Crear directorio principal
mkdir -p ~/projects/print
cd ~/projects/print

# Crear subdirectorios necesarios
mkdir -p cups-config       # Configuración de CUPS (no usado finalmente)
mkdir -p print-output      # Archivos "impresos" por CUPS
mkdir -p client-files      # Archivos del cliente y capturas
mkdir -p wireshark-config  # Configuración de Wireshark web
mkdir -p captures          # Capturas adicionales
```

### Estructura final del proyecto:
```
~/projects/print/
├── docker-compose.yaml      # Configuración de los contenedores
├── capture-lpd.sh          # Script de automatización (opcional)
├── cups-config/            # Configuración CUPS (vacío)
├── print-output/           # Salida de impresión (vacío inicialmente)
├── client-files/           # Archivos de prueba y capturas
│   ├── test.txt           # Archivo a "imprimir"
│   └── lpd-capture.pcap   # Captura de tráfico
├── wireshark-config/       # Config Wireshark web
└── captures/              # Capturas adicionales
```

---

## 3. Configuración del Entorno {#configuración-del-entorno}

### Archivo docker-compose.yaml

Crear el archivo principal de configuración:

```bash
cat > docker-compose.yaml << 'EOF'
services:
  # ============================================
  # SERVIDOR CUPS CON SOPORTE LPD
  # ============================================
  cups-server:
    image: olbat/cupsd                    # Imagen Docker con CUPS preinstalado
    container_name: cups-server
    hostname: cups-server
    ports:
      - "631:631"                         # Puerto web de CUPS (administración)
      - "515:515"                         # Puerto LPD (Line Printer Daemon)
    environment:
      - ADMIN_PASSWORD=admin              # Contraseña del administrador de CUPS
    volumes:
      - ./print-output:/tmp/prints        # Carpeta donde se guardarán los archivos "impresos"
    networks:
      printer-net:
        ipv4_address: 172.20.0.10         # IP fija para el servidor
    cap_add:
      - NET_ADMIN                         # Permisos de red necesarios
    privileged: true                      # Necesario para que CUPS funcione correctamente

  # ============================================
  # CLIENTE LPD (Ubuntu con herramientas)
  # ============================================
  lpd-client:
    image: ubuntu:22.04                   # Ubuntu base
    container_name: lpd-client
    hostname: lpd-client
    tty: true                             # Mantener terminal activa
    stdin_open: true                      # Permitir entrada interactiva
    networks:
      printer-net:
        ipv4_address: 172.20.0.20         # IP fija para el cliente
    cap_add:
      - NET_ADMIN                         # Necesario para tcpdump
      - NET_RAW                           # Necesario para captura de paquetes
    volumes:
      - ./client-files:/files             # Carpeta compartida con el host
    depends_on:
      - cups-server                       # Espera a que el servidor esté listo
    # Comando que instala herramientas necesarias al iniciar el contenedor
    command: bash -c "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y cups-bsd tcpdump tshark iputils-ping netcat-openbsd && echo 'Paquetes instalados' && tail -f /dev/null"

  # ============================================
  # WIRESHARK WEB (OPCIONAL)
  # ============================================
  wireshark:
    image: lscr.io/linuxserver/wireshark:latest  # Wireshark con interfaz web
    container_name: wireshark
    hostname: wireshark
    environment:
      - PUID=1000                         # ID de usuario
      - PGID=1000                         # ID de grupo
      - TZ=Europe/Madrid                  # Zona horaria
    ports:
      - "3000:3000"                       # Puerto para acceder vía navegador
    networks:
      printer-net:
        ipv4_address: 172.20.0.30         # IP fija para Wireshark
    cap_add:
      - NET_ADMIN                         # Necesario para captura
      - NET_RAW                           # Necesario para captura
    security_opt:
      - seccomp:unconfined                # Permisos de seguridad necesarios
    volumes:
      - ./wireshark-config:/config        # Configuración persistente
      - ./captures:/captures              # Capturas guardadas

# ============================================
# RED INTERNA
# ============================================
networks:
  printer-net:
    driver: bridge                        # Red tipo bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16           # Rango de IPs para la red
EOF
```

**Explicación de componentes:**

- **cups-server**: Servidor de impresión CUPS que actuará como servidor LPD
- **lpd-client**: Cliente Ubuntu con herramientas para enviar trabajos y capturar tráfico
- **wireshark**: Interfaz web de Wireshark (opcional, para análisis en navegador)
- **printer-net**: Red privada donde se comunican los contenedores

**Paquetes instalados en el cliente:**
- `cups-bsd`: Cliente LPD (incluye comandos `lpr`, `lpq`, `lprm`)
- `tcpdump`: Captura de paquetes de red
- `tshark`: Versión CLI de Wireshark para análisis
- `iputils-ping`: Herramienta ping para verificar conectividad
- `netcat-openbsd`: Herramienta para conexiones TCP/UDP directas

---

## 4. Puesta en Marcha {#puesta-en-marcha}

### Paso 1: Iniciar los contenedores

```bash
# Iniciar todos los servicios
docker compose up -d

# Verificar que están corriendo
docker ps

# Deberías ver 3 contenedores: cups-server, lpd-client, wireshark
```

**Resultado esperado:**
```
CONTAINER ID   IMAGE                                    STATUS
8551a20ef73d   olbat/cupsd                             Up 2 minutes
a1b2c3d4e5f6   ubuntu:22.04                            Up 2 minutes
f6e5d4c3b2a1   lscr.io/linuxserver/wireshark:latest   Up 2 minutes
```

### Paso 2: Esperar instalación del cliente

```bash
# El cliente tarda ~45 segundos en instalar los paquetes la primera vez
# Ver progreso:
docker logs -f lpd-client

# Presiona Ctrl+C cuando veas "Paquetes instalados"
```

### Paso 3: Configurar la impresora en CUPS

```bash
# Entrar al contenedor del servidor
docker exec -it cups-server sh

# Dentro del contenedor, ejecutar:
apt-get update
apt-get install -y cups-bsd socat

# Salir del contenedor
exit
```

Ahora configurar la impresora:

```bash
# Crear impresora virtual que escribe a un archivo
docker exec cups-server sh -c "
  echo 'FileDevice Yes' >> /etc/cups/cups-files.conf
  cupsctl --share-printers
  cupsctl --remote-admin
  lpadmin -p test-printer -E -v file:/tmp/prints/output.txt -m drv:///sample.drv/generic.ppd
  lpstat -v
"
```

**Resultado esperado:**
```
lpadmin: Printer drivers are deprecated and will stop working in a future version of CUPS.
device for test-printer: /tmp/prints/output.txt
```

**Explicación:**
- `FileDevice Yes`: Permite que CUPS escriba en archivos (por defecto está deshabilitado)
- `cupsctl --share-printers`: Comparte impresoras en la red
- `cupsctl --remote-admin`: Permite administración remota
- `lpadmin -p test-printer`: Crea una impresora llamada "test-printer"
- `-v file:/tmp/prints/output.txt`: El "dispositivo" es un archivo
- El warning sobre deprecated drivers es normal y no afecta

### Paso 4: Iniciar el servicio LPD con socat

El daemon LPD necesita estar escuchando en el puerto 515:

```bash
# Iniciar cups-lpd en el puerto 515 usando socat
docker exec -d cups-server socat TCP-LISTEN:515,fork,reuseaddr EXEC:"/usr/lib/cups/daemon/cups-lpd -o document-format=application/octet-stream"

# Verificar que el puerto está abierto
docker exec lpd-client nc -zv cups-server 515
```

**Resultado esperado:**
```
cups-server (172.20.0.10:515) open
```

**Explicación de socat:**
- `TCP-LISTEN:515`: Escucha en el puerto 515
- `fork`: Crea un proceso hijo para cada conexión
- `reuseaddr`: Permite reutilizar la dirección inmediatamente
- `EXEC`: Ejecuta cups-lpd para cada conexión entrante

---

## 5. Captura y Análisis del Tráfico LPD {#captura-y-análisis}

### Paso 1: Crear archivo de prueba

```bash
# Crear un archivo de texto para "imprimir"
echo "Test LPD - $(date)" | docker exec -i lpd-client tee /files/test.txt
```

### Paso 2: Iniciar captura de paquetes

```bash
# Iniciar tcpdump en el cliente capturando tráfico del puerto 515
docker exec -d lpd-client tcpdump -i eth0 -w /files/lpd-capture.pcap port 515

# Esperar 2 segundos para que tcpdump inicie
sleep 2
```

**Explicación:**
- `-d`: Ejecuta en background (daemon)
- `-i eth0`: Captura en la interfaz de red eth0
- `-w /files/lpd-capture.pcap`: Guarda la captura en este archivo
- `port 515`: Solo captura tráfico del puerto LPD

### Paso 3: Enviar trabajo de impresión usando protocolo LPD

```bash
# Enviar trabajo usando el protocolo LPD manualmente con netcat
docker exec lpd-client sh -c '
HOSTNAME="lpd-client"              # Nombre del host cliente
JOBID="001"                        # ID único del trabajo
QUEUE="test-printer"               # Nombre de la cola de impresión
DATAFILE="/files/test.txt"         # Archivo a imprimir

# Crear archivo de control con metadatos del trabajo
CONTROL="H${HOSTNAME}
Proot
Jtest-job
ldfA${JOBID}${HOSTNAME}
UdfA${JOBID}${HOSTNAME}
Ntest.txt
"

# Calcular tamaños
CONTROL_SIZE=$(echo -n "$CONTROL" | wc -c)
DATA_SIZE=$(wc -c < $DATAFILE)

# Enviar comandos LPD al servidor
(
  # Comando 02: Solicitar recibir un trabajo en la cola
  printf "\x02${QUEUE}\n"
  sleep 0.1
  
  # Comando 02: Enviar archivo de control
  printf "\x02${CONTROL_SIZE} cfA${JOBID}${HOSTNAME}\n"
  echo -n "$CONTROL"
  printf "\x00"                    # Byte nulo de terminación
  sleep 0.1
  
  # Comando 03: Enviar archivo de datos
  printf "\x03${DATA_SIZE} dfA${JOBID}${HOSTNAME}\n"
  cat $DATAFILE
  printf "\x00"                    # Byte nulo de terminación
) | nc cups-server 515             # Enviar todo al puerto 515

echo "Trabajo enviado a LPD"
'
```

**Explicación del protocolo LPD:**

**Comandos principales:**
- `\x02` (02 hex): Receive printer job / Receive control file
- `\x03` (03 hex): Receive data file
- `\x00` (00 hex): Byte nulo de terminación

**Archivo de control (cfA):**
- `H`: Nombre del host que envía
- `P`: Nombre de usuario
- `J`: Nombre del trabajo
- `l`: Imprimir archivo literal (letra ele minúscula)
- `U`: Unlink (borrar después de imprimir)
- `N`: Nombre del archivo original

**Nomenclatura de archivos:**
- `cfA001lpd-client`: Control File, Job A, ID 001, host lpd-client
- `dfA001lpd-client`: Data File, Job A, ID 001, host lpd-client

### Paso 4: Detener captura

```bash
# Esperar a que termine la transmisión
sleep 5

# Detener tcpdump
docker exec lpd-client pkill tcpdump

# Copiar captura al directorio actual del host
cp client-files/lpd-capture.pcap ./

# Ver tamaño de la captura
ls -lh lpd-capture.pcap
```

**Resultado esperado:**
```
-rw-r--r-- 1 elorri elorri 1.2K nov 17 21:13 lpd-capture.pcap
```

### Paso 5: Analizar la captura

#### Opción 1: Con tshark (línea de comandos)

```bash
# Ver todos los paquetes capturados
docker exec lpd-client tshark -r /files/lpd-capture.pcap -Y "tcp.port==515" 2>/dev/null
```

**Resultado esperado:**
```
    1   0.000000  172.20.0.20 → 172.20.0.10  TCP 74 48694 → 515 [SYN]
    2   0.000032  172.20.0.10 → 172.20.0.20  TCP 74 515 → 48694 [SYN, ACK]
    3   0.000042  172.20.0.20 → 172.20.0.10  TCP 66 48694 → 515 [ACK]
    4   0.000092  172.20.0.20 → 172.20.0.10  LPD 83 LPD continuation
    5   0.000100  172.20.0.10 → 172.20.0.20  TCP 66 515 → 48694 [ACK]
    6   0.003564  172.20.0.10 → 172.20.0.20  LPD 67 LPD response
    ...
```

#### Opción 2: Con Wireshark (interfaz gráfica)

```bash
# Abrir Wireshark con la captura
wireshark lpd-capture.pcap &
```

En Wireshark:
1. Aplicar filtro: `tcp.port == 515`
2. Click derecho en cualquier paquete → Follow → TCP Stream
3. Verás el diálogo completo del protocolo LPD

---

## 6. Interpretación de Resultados {#interpretación-de-resultados}

### Contenido capturado del TCP Stream:

```
\x02test-printer
.
\x0274 cfA001lpd-client
Hlpd-client
Proot
Jtest-job
ldfA001lpd-client
UdfA001lpd-client
Ntest.txt
\x00\x0340 dfA001lpd-client
Test LPD - lun 17 nov 2025 21:00:31 CET
\x00
```

### Desglose del tráfico:

1. **Primera línea: `\x02test-printer`**
   - Comando 02: "Quiero enviar un trabajo"
   - Cola destino: "test-printer"

2. **Archivo de control: `\x0274 cfA001lpd-client`**
   - Comando 02: "Aquí va el archivo de control"
   - Tamaño: 74 bytes
   - Nombre: cfA001lpd-client

3. **Contenido del archivo de control:**
   ```
   Hlpd-client          → Host origen
   Proot                → Usuario
   Jtest-job            → Nombre del trabajo
   ldfA001lpd-client    → Imprimir archivo de datos
   UdfA001lpd-client    → Borrar después de imprimir
   Ntest.txt            → Nombre del archivo original
   ```

4. **Archivo de datos: `\x0340 dfA001lpd-client`**
   - Comando 03: "Aquí van los datos"
   - Tamaño: 40 bytes
   - Nombre: dfA001lpd-client

5. **Contenido del archivo de datos:**
   ```
   Test LPD - lun 17 nov 2025 21:00:31 CET
   ```

6. **Bytes nulos (`\x00`)**: Finalizan cada sección

### Flujo de comunicación TCP:

1. **Handshake TCP** (paquetes 1-3): Establecimiento de conexión
2. **Envío de comando** (paquete 4): Cliente solicita enviar trabajo
3. **ACK del servidor** (paquetes 5-6): Servidor acepta
4. **Envío de control** (paquetes 10-11): Cliente envía metadatos
5. **Envío de datos** (paquete 11): Cliente envía contenido
6. **Cierre** (paquete 12): Servidor cierra conexión

---

## 7. Script de Automatización {#script-de-automatización}

Para repetir fácilmente el experimento, puedes usar este script:

```bash
cat > capture-lpd.sh << 'SCRIPT'
#!/bin/bash
# ====================================================
# Script automatizado para captura de tráfico LPD
# ====================================================

set -e  # Salir si hay error

# Colores
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "=== Script de captura de tráfico LPD ==="
echo ""

# 1. Verificar contenedores
echo -e "${YELLOW}1. Verificando contenedores...${NC}"
if ! docker ps | grep -q cups-server; then
    echo -e "${RED}Error: cups-server no está corriendo${NC}"
    exit 1
fi
if ! docker ps | grep -q lpd-client; then
    echo -e "${RED}Error: lpd-client no está corriendo${NC}"
    exit 1
fi
echo -e "${GREEN}✓ Contenedores activos${NC}"
echo ""

# 2. Verificar puerto 515
echo -e "${YELLOW}2. Verificando puerto 515...${NC}"
if docker exec lpd-client nc -zv cups-server 515 2>&1 | grep -q "succeeded"; then
    echo -e "${GREEN}✓ Puerto 515 accesible${NC}"
else
    echo -e "${RED}Error: Puerto 515 no accesible${NC}"
    exit 1
fi
echo ""

# 3. Limpiar capturas anteriores
echo -e "${YELLOW}3. Limpiando capturas anteriores...${NC}"
docker exec lpd-client pkill tcpdump 2>/dev/null || true
rm -f client-files/lpd-capture.pcap lpd-capture.pcap
echo -e "${GREEN}✓ Limpieza completada${NC}"
echo ""

# 4. Crear archivo de prueba
echo -e "${YELLOW}4. Creando archivo de prueba...${NC}"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
echo "Test LPD - $TIMESTAMP" | docker exec -i lpd-client tee /files/test.txt > /dev/null
echo -e "${GREEN}✓ Archivo creado${NC}"
echo ""

# 5. Iniciar captura
echo -e "${YELLOW}5. Iniciando captura...${NC}"
docker exec -d lpd-client tcpdump -i eth0 -w /files/lpd-capture.pcap port 515
sleep 2
echo -e "${GREEN}✓ Captura en progreso${NC}"
echo ""

# 6. Enviar trabajo LPD
echo -e "${YELLOW}6. Enviando trabajo de impresión...${NC}"
docker exec lpd-client sh -c '
HOSTNAME="lpd-client"
JOBID="001"
QUEUE="test-printer"
DATAFILE="/files/test.txt"

CONTROL="H${HOSTNAME}
Proot
Jtest-job
ldfA${JOBID}${HOSTNAME}
UdfA${JOBID}${HOSTNAME}
Ntest.txt
"

CONTROL_SIZE=$(echo -n "$CONTROL" | wc -c)
DATA_SIZE=$(wc -c < $DATAFILE)

(
  printf "\x02${QUEUE}\n"
  sleep 0.1
  printf "\x02${CONTROL_SIZE} cfA${JOBID}${HOSTNAME}\n"
  echo -n "$CONTROL"
  printf "\x00"
  sleep 0.1
  printf "\x03${DATA_SIZE} dfA${JOBID}${HOSTNAME}\n"
  cat $DATAFILE
  printf "\x00"
) | nc cups-server 515
' > /dev/null 2>&1
echo -e "${GREEN}✓ Trabajo enviado${NC}"
echo ""

# 7. Detener captura
echo -e "${YELLOW}7. Finalizando captura...${NC}"
sleep 3
docker exec lpd-client pkill tcpdump 2>/dev/null || true
sleep 1
echo -e "${GREEN}✓ Captura finalizada${NC}"
echo ""

# 8. Copiar y analizar
echo -e "${YELLOW}8. Analizando resultados...${NC}"
cp client-files/lpd-capture.pcap ./
FILESIZE=$(ls -lh lpd-capture.pcap | awk '{print $5}')
PACKETS=$(docker exec lpd-client tshark -r /files/lpd-capture.pcap 2>/dev/null | wc -l)
LPD_PACKETS=$(docker exec lpd-client tshark -r /files/lpd-capture.pcap -Y "lpd" 2>/dev/null | wc -l)

echo -e "${GREEN}✓ Archivo: lpd-capture.pcap (${FILESIZE})${NC}"
echo -e "${GREEN}✓ Paquetes totales: ${PACKETS}${NC}"
echo -e "${GREEN}✓ Paquetes LPD: ${LPD_PACKETS}${NC}"
echo ""

# 9. Mostrar primeros paquetes
echo -e "${YELLOW}=== Primeros paquetes capturados ===${NC}"
docker exec lpd-client tshark -r /files/lpd-capture.pcap -Y "tcp.port==515" 2>/dev/null | head -12
echo ""

# 10. Instrucciones
echo -e "${GREEN}=== ¡Captura completada! ===${NC}"
echo ""
echo "Para analizar:"
echo -e "  ${YELLOW}wireshark lpd-capture.pcap${NC}"
echo ""
echo "En Wireshark:"
echo "  1. Filtro: tcp.port == 515"
echo "  2. Click derecho → Follow → TCP Stream"
echo ""
SCRIPT

# Hacer ejecutable
chmod +x capture-lpd.sh
```

### Uso del script:

```bash
# Ejecutar
./capture-lpd.sh
```

---

## Resumen del Experimento

### Lo que hemos logrado:

✅ **Entorno completo con Docker**
- Servidor CUPS configurado con soporte LPD
- Cliente Ubuntu con herramientas de captura
- Red aislada para análisis seguro

✅ **Protocolo LPD capturado**
- Tráfico completo entre cliente y servidor
- Comandos LPD visibles y analizables
- Metadatos y datos transmitidos

✅ **Análisis con Wireshark**
- Paquetes correctamente etiquetados como LPD
- TCP stream completo visible
- Protocolo completamente decodificado

### Comandos clave aprendidos:

```bash
# Gestión de contenedores
docker compose up -d              # Iniciar servicios
docker compose down               # Detener servicios
docker ps                         # Ver contenedores activos
docker exec -it <nombre> bash     # Entrar a un contenedor
docker logs <nombre>              # Ver logs

# Captura de paquetes
tcpdump -i eth0 -w file.pcap port 515   # Capturar tráfico
tshark -r file.pcap -Y "lpd"            # Analizar captura
wireshark file.pcap                      # Analizar gráficamente

# Protocolo LPD
\x02<cola>           # Receive job
\x02<size> cfA...    # Send control file
\x03<size> dfA...    # Send data file
\x00                 # Null terminator
```

### Archivos importantes generados:

- `docker-compose.yaml`: Configuración de contenedores
- `lpd-capture.pcap`: Captura del tráfico LPD
- `capture-lpd.sh`: Script de automatización
- `client-files/test.txt`: Archivo de prueba