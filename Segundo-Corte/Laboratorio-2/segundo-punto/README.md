# Laboratorio 2 — Punto 2: Red Física, YOLO, Wireshark e iPerf3
**Asignatura:** Conmutación y Teletráfico  
**Docente:** Diego Alejandro Barragán Vargas — Fundación Universitaria Compensar

---

## Tabla de Contenidos
1. [Resumen de infraestructura](#1-resumen-de-infraestructura)
2. [Topología en GNS3](#2-topología-en-gns3)
3. [Paso 1 — Configuración del Switch Capa 2 (IOU L2)](#3-paso-1--configuración-del-switch-capa-2-iou-l2)
4. [Paso 2 — Configuración del Router Capa 3 (c3725)](#4-paso-2--configuración-del-router-capa-3-c3725)
5. [Paso 3 — VM Servidor (192.168.1.10)](#5-paso-3--vm-servidor-192168110)
6. [Paso 4 — VM Cliente Monitor (192.168.1.20)](#6-paso-4--vm-cliente-monitor-192168120)
7. [Paso 5 — VM Cliente Tráfico (192.168.1.30)](#7-paso-5--vm-cliente-tráfico-192168130)
8. [Paso 6 — Script YOLO en el Servidor](#8-paso-6--script-yolo-en-el-servidor)
9. [Paso 7 — Generación de tráfico con iPerf3](#9-paso-7--generación-de-tráfico-con-iperf3)
10. [Paso 8 — Captura y análisis con Wireshark GUI](#10-paso-8--captura-y-análisis-con-wireshark-gui)
11. [Paso 9 — Limitación de ancho de banda en el Router (QoS)](#11-paso-9--limitación-de-ancho-de-banda-en-el-router-qos)
12. [Paso 10 — Integración con NetFlow/sFlow (conexión con Punto 1)](#12-paso-10--integración-con-netflowsflow-conexión-con-punto-1)
13. [Respuestas a preguntas del laboratorio](#13-respuestas-a-preguntas-del-laboratorio)

---

## 1. Resumen de infraestructura

### Descripción del escenario

El segundo punto correlaciona el mundo físico con el tráfico de red. Una VM servidor emite un flujo de video RTSP y ejecuta detección de objetos con YOLO. Dos VMs clientes cumplen roles distintos: el Cliente Monitor captura y analiza tráfico visualmente con **Wireshark GUI**, mientras que el Cliente Tráfico genera carga de red con iPerf3. Todo el tráfico pasa por un Switch de Capa 2 (IOU L2) y un Router de Capa 3 (c3725) configurados en GNS3.

Se usa **Ubuntu Desktop** en las tres VMs para poder ejecutar Wireshark con interfaz gráfica, lo que permite analizar visualmente los protocolos RTSP, RTP, TCP y UDP tal como lo requiere el laboratorio.

### Decisión de hardware virtual

| Dispositivo | Implementación | Justificación |
|---|---|---|
| Switch Capa 2 | IOU L2 en GNS3 | Mismo que Punto 1, ya operativo |
| Router Capa 3 | c3725 en GNS3 | Liviano, no requiere KVM, soporta `policy-map` para QoS |
| Servidor | Ubuntu Desktop VM (VirtualBox) | Necesita GUI para ver output YOLO y tiene ffmpeg nativo |
| Cliente Monitor | Ubuntu Desktop VM (VirtualBox) | **Wireshark GUI** para análisis visual de protocolos |
| Cliente Tráfico | Ubuntu Desktop VM (VirtualBox) | Consistente con el resto del escenario |

### Adaptadores Host-Only de VirtualBox

| Adaptador Windows | Host-Only VirtualBox | IP del Host Windows | Uso |
|---|---|---|---|
| Ethernet 9 | Adapter #6 | 192.168.1.1/24 | Cloud-P2 → red 192.168.1.0/24 |
| Ethernet 10 | Adapter #7 | 10.0.0.1/24 | Cloud-EXT → subred externa (opcional) |

> Usa adaptadores Host-Only distintos a los del Punto 1.

### Tabla de IPs

| VM | IP | Gateway | Rol |
|---|---|---|---|
| Servidor | 192.168.1.10/24 | 192.168.1.1 | RTSP + YOLO + iPerf3 server |
| Cliente Monitor | 192.168.1.20/24 | 192.168.1.1 | Wireshark GUI + consumidor RTSP |
| Cliente Tráfico | 192.168.1.30/24 | 192.168.1.1 | iPerf3 cliente |
| Router c3725 Fa0/0 | 192.168.1.1/24 | — | Gateway de la LAN |
| Router c3725 Fa0/1 | 10.0.0.1/24 | — | Subred externa (opcional) |

### Configuración de adaptadores en cada VM (VirtualBox)

Todas las VMs del Punto 2 tienen la misma estructura de adaptadores:

| Adaptador VirtualBox | Tipo | Interfaz Linux | Propósito |
|---|---|---|---|
| Adaptador 1 | NAT | `enp0s3` | Internet (actualizaciones, pip) |
| Adaptador 2 | Host-Only #6 | `enp0s8` | Red 192.168.1.0/24 del laboratorio |

---

## 2. Topología en GNS3

```
    VirtualBox (Windows 11) — Host-Only Adapter #6 — 192.168.1.0/24
    ┌──────────────────────────────────────────────────────────────┐
    │  Servidor VM           Monitor VM          Tráfico VM        │
    │  Ubuntu Desktop        Ubuntu Desktop      Ubuntu Desktop    │
    │  192.168.1.10          192.168.1.20        192.168.1.30      │
    │  RTSP + YOLO           Wireshark GUI       iPerf3 cliente    │
    │  iPerf3 server         VLC / ffplay                          │
    └──────┬─────────────────────┬──────────────────┬─────────────┘
           │                     │                  │
     enp0s8 (Host-Only Adapter #6 — Ethernet 9 Windows)
           │
      Cloud-P2 (GNS3)
           │
         e0/0
    ┌──────────────┐
    │   SW-L2-P2   │  ← IOU L2
    └──────┬───────┘
         e0/1
    ┌──────────────────────┐
    │    Router c3725      │
    │  Fa0/0: 192.168.1.1  │  ← Gateway LAN + QoS
    │  Fa0/1: 10.0.0.1     │  ← Subred externa
    └──────────────────────┘
         Fa0/1
           │
      Cloud-EXT (Ethernet 10 — opcional)

Flujos de tráfico:
  Servidor → Switch → Monitor VM    : RTSP (TCP 554) + RTP (UDP alto)
  Tráfico VM → Switch → Servidor    : iPerf3 TCP/UDP (puerto 5201)
  Todos → Switch → Router           : tráfico enrutado
  Wireshark captura en              : enp0s8 del Cliente Monitor
```

### Nodos en GNS3

| Nodo GNS3 | Tipo | Interfaz Windows |
|---|---|---|
| SW-L2-P2 | Switch IOU L2 | — |
| Router-L3-P2 | Router c3725 | — |
| Cloud-P2 | Cloud node | Ethernet 9 (Adapter #6) |
| Cloud-EXT | Cloud node | Ethernet 10 (Adapter #7, opcional) |

---

## 3. Paso 1 — Configuración del Switch Capa 2 (IOU L2)

Hacer clic derecho sobre SW-L2-P2 en GNS3 → Console.

```cisco
enable
configure terminal

! === VLAN de video ===
vlan 100
 name VLAN_VIDEO
exit

! === Puerto hacia la red Host-Only (Cloud-P2) ===
interface ethernet 0/0
 description Red-192.168.1.0-VMs
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
exit

! === Puerto uplink hacia el Router c3725 ===
interface ethernet 0/1
 description Uplink-Router-c3725
 switchport mode access
 switchport access vlan 100
 no shutdown
exit

! === IP de gestión ===
interface vlan 100
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1

end
write memory
```

### Verificación

```cisco
show vlan brief
show interfaces status
show running-config
```

---

## 4. Paso 2 — Configuración del Router Capa 3 (c3725)

Hacer clic derecho sobre Router-L3-P2 en GNS3 → Console.

```cisco
enable
configure terminal

! === Interfaz LAN — gateway de 192.168.1.0/24 ===
interface FastEthernet0/0
 description LAN-hacia-Switch-L2
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit

! === Interfaz externa opcional ===
interface FastEthernet0/1
 description Subred-Externa
 ip address 10.0.0.1 255.255.255.0
 no shutdown
exit

! === Habilitar routing entre interfaces ===
ip routing

! === SNMP para monitoreo opcional desde Zabbix del Punto 1 ===
snmp-server community public RO
snmp-server community private RW
snmp-server location "Laboratorio_P2"
snmp-server contact "admin@lab.com"

end
write memory
```

### Verificación

```cisco
show ip interface brief
show ip route
show running-config
```

---

## 5. Paso 3 — VM Servidor (192.168.1.10)

**Configuración VirtualBox:**
- Adaptador 1: NAT → `enp0s3`
- Adaptador 2: Host-Only Adapter #6 → `enp0s8`
- SO recomendado: Ubuntu Desktop 22.04

### 5.1 Configurar IP estática

```bash
sudo tee /etc/netplan/01-lab-p2.yaml <<EOF
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.1.10/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
EOF

sudo netplan apply

# Verificar
ip addr show enp0s8
ping -c 3 192.168.1.1
```

### 5.2 Instalar dependencias

```bash
sudo apt update
sudo apt install -y \
  python3 python3-pip python3-venv \
  ffmpeg \
  iperf3 \
  net-tools wget curl

# Entorno virtual para YOLO
python3 -m venv ~/yolo_env
source ~/yolo_env/bin/activate
pip install opencv-python ultralytics

# Verificar instalación
python3 -c "from ultralytics import YOLO; print('YOLO OK')"
```

### 5.3 Levantar el servidor RTSP con ffmpeg

```bash
# Descargar video de prueba
wget -O /tmp/test_video.mp4 \
  "https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4"

# Lanzar stream RTSP en loop continuo
ffmpeg -re -stream_loop -1 -i /tmp/test_video.mp4 \
  -vcodec libx264 -preset ultrafast -tune zerolatency \
  -f rtsp rtsp://0.0.0.0:554/stream &

# Verificar puerto activo
sudo ss -tlnp | grep 554
```

### 5.4 Iniciar servidor iPerf3

```bash
iperf3 -s -p 5201 -D
sudo ss -tlnp | grep 5201
```

---

## 6. Paso 4 — VM Cliente Monitor (192.168.1.20)

Esta VM es el centro de análisis del laboratorio. Ejecuta **Wireshark con interfaz gráfica** para examinar visualmente todos los protocolos: RTSP, RTP, iPerf3 TCP/UDP.

**Configuración VirtualBox:**
- Adaptador 1: NAT → `enp0s3`
- Adaptador 2: Host-Only Adapter #6 → `enp0s8`
- SO: Ubuntu Desktop 22.04 (**con entorno gráfico**)

### 6.1 Configurar IP estática

```bash
sudo tee /etc/netplan/01-lab-p2.yaml <<EOF
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.1.20/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
EOF

sudo netplan apply

ping -c 3 192.168.1.10
```

### 6.2 Instalar Wireshark y VLC

```bash
sudo apt update
sudo apt install -y wireshark vlc

# Permitir captura sin root
sudo dpkg-reconfigure wireshark-common    # Seleccionar Sí
sudo usermod -aG wireshark $USER

# Cerrar sesión y volver a entrar para aplicar el grupo
# o ejecutar:
newgrp wireshark
```

### 6.3 Consumir el stream RTSP (genera tráfico observable)

```bash
# Abrir el stream RTSP desde la interfaz gráfica de VLC:
# Media → Open Network Stream → rtsp://192.168.1.10:554/stream

# O desde terminal (en background)
vlc rtsp://192.168.1.10:554/stream &
```

---

## 7. Paso 5 — VM Cliente Tráfico (192.168.1.30)

**Configuración VirtualBox:**
- Adaptador 1: NAT → `enp0s3`
- Adaptador 2: Host-Only Adapter #6 → `enp0s8`
- SO: Ubuntu Desktop 22.04

### 7.1 Configurar IP estática

```bash
sudo tee /etc/netplan/01-lab-p2.yaml <<EOF
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.1.30/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
EOF

sudo netplan apply

ping -c 3 192.168.1.10
```

### 7.2 Instalar iPerf3

```bash
sudo apt update && sudo apt install -y iperf3
```

---

## 8. Paso 6 — Script YOLO en el Servidor

El script lee el flujo RTSP local, aplica detección de objetos con YOLOv8 y muestra los resultados visualmente en la pantalla del Servidor. También imprime estadísticas de FPS para correlacionar con lo que captura Wireshark en el Monitor.

El código fuente está en el repositorio del docente:
```
https://github.com/dialejobv/conmutacion_teletrafico/tree/main/2)%20LABORATORIO
```

### 8.1 Descargar el script

```bash
# En la VM Servidor, con el entorno virtual activo
source ~/yolo_env/bin/activate

wget -O ~/conmutacion.py \
  "https://raw.githubusercontent.com/dialejobv/conmutacion_teletrafico/main/2)%20LABORATORIO/conmutacion.py"
```

### 8.2 Ejecutar YOLO sobre el flujo RTSP

```bash
source ~/yolo_env/bin/activate

# Modelo nano — bajo consumo, ideal para observar FPS estable primero
python3 ~/conmutacion.py \
  --source rtsp://127.0.0.1:554/stream \
  --model yolov8n.pt \
  --output /tmp/yolo_output.mp4
```

La ventana de YOLO muestra el video con los bounding boxes de los objetos detectados en tiempo real. Los FPS se imprimen en consola cada segundo.

### 8.3 Comparativa de modelos para el análisis de QoS

| Modelo | Tamaño | FPS típico (CPU VM) | Efecto en tráfico |
|---|---|---|---|
| `yolov8n.pt` | Nano ~6 MB | 20-30 FPS | Mínimo — referencia base |
| `yolov8s.pt` | Small ~22 MB | 12-18 FPS | Moderado |
| `yolov8l.pt` | Large ~87 MB | 2-5 FPS | Alto — CPU saturada, ffmpeg pierde frames |

> **Prueba recomendada:** Correr primero con `yolov8n.pt` y capturar en Wireshark. Luego cambiar a `yolov8l.pt` y comparar el jitter RTP y la tasa de bits en el gráfico IO de Wireshark.

---

## 9. Paso 7 — Generación de tráfico con iPerf3

Ejecutar desde la **VM Cliente Tráfico** mientras YOLO está corriendo en el Servidor y Wireshark captura en el Monitor.

### 9.1 Prueba TCP baseline

```bash
# VM Cliente Tráfico
iperf3 -c 192.168.1.10 -t 60 -i 1 -P 4
```

### 9.2 Prueba UDP 10 Mbps

```bash
iperf3 -c 192.168.1.10 -u -b 10M -t 60 -i 1
```

### 9.3 Prueba UDP 50 Mbps (impacto máximo en YOLO)

```bash
# Observar en tiempo real cómo caen los FPS de YOLO en el Servidor
iperf3 -c 192.168.1.10 -u -b 50M -t 60 -i 1
```

### 9.4 Tabla comparativa de pruebas

| Prueba | Comando | Qué observar en Wireshark | Efecto en YOLO FPS |
|---|---|---|---|
| TCP baseline | `-t 60 -P 4` | Handshake SYN/SYN-ACK/ACK, ventana TCP crece | Sin impacto |
| UDP 10 Mbps | `-u -b 10M -t 60` | Flujo UDP constante, pérdida mínima | Sin impacto notable |
| UDP 50 Mbps | `-u -b 50M -t 60` | Pérdida de paquetes RTP, jitter aumenta | Caída de FPS visible |
| UDP 50 Mbps + QoS | Igual + rate-limit activo | Tráfico total limitado a ~10 Mbps | FPS se estabiliza |

---

## 10. Paso 8 — Captura y análisis con Wireshark GUI

Todos los pasos siguientes se realizan en la **VM Cliente Monitor (192.168.1.20)** con Wireshark abierto en modo gráfico.

### 10.1 Abrir Wireshark y seleccionar la interfaz

1. Abrir Wireshark desde el menú de aplicaciones o ejecutar `wireshark &` en terminal.
2. En la pantalla de inicio, seleccionar la interfaz **`enp0s8`** (la que tiene tráfico).
3. Hacer clic en el botón azul de la aleta de tiburón para iniciar la captura.

### 10.2 Captura general — filtro por IP del servidor

En la barra de filtro de Wireshark escribir:

```
ip.addr == 192.168.1.10
```

Esto muestra solo el tráfico hacia y desde el Servidor. Se verán los paquetes RTSP, RTP e iPerf3 mezclados. Aplicar luego filtros específicos por protocolo.

### 10.3 Análisis del protocolo RTSP (TCP 554)

**Filtro en Wireshark:**
```
rtsp
```

Se observará la sesión completa de negociación:

| Mensaje RTSP | Dirección | Qué indica |
|---|---|---|
| `DESCRIBE` | Monitor → Servidor | Cliente solicita descripción del stream |
| `200 OK` + SDP | Servidor → Monitor | Respuesta con codec H.264 y puertos RTP |
| `SETUP` | Monitor → Servidor | Cliente negocia puertos UDP para RTP |
| `PLAY` | Monitor → Servidor | Inicio del stream de video |
| `TEARDOWN` | Monitor → Servidor | Cierre de la sesión |

> **Captura de pantalla recomendada:** Seleccionar un paquete DESCRIBE y expandir el panel inferior para mostrar el contenido SDP con los parámetros del stream.

### 10.4 Análisis del protocolo RTP (UDP puertos altos)

**Filtro en Wireshark:**
```
rtp
```

Para ver los campos clave del encabezado RTP hacer clic en cualquier paquete RTP y expandir en el panel inferior la sección **Real-Time Transport Protocol**:

- **Sequence number:** Debe incrementar en 1 por paquete. Un salto indica pérdida.
- **Timestamp:** Refleja el tiempo del fotograma en el servidor. Aumenta uniformemente.
- **SSRC:** Identificador único del stream.

**Análisis de stream RTP con gráficos:**

1. Ir a **Telephony → RTP → RTP Streams**
2. Seleccionar el stream del Servidor
3. Hacer clic en **Analyze**
4. Se abrirá una ventana con:
   - Gráfico de **jitter** en el tiempo
   - Contador de **paquetes perdidos**
   - Delta entre paquetes

> **Captura de pantalla recomendada:** La ventana de RTP Stream Analysis mostrando el gráfico de jitter antes y después de lanzar iPerf3 a 50 Mbps.

### 10.5 Análisis de iPerf3 en modo TCP

**Filtro en Wireshark:**
```
tcp.port == 5201
```

Seleccionar el stream TCP y usar **Statistics → TCP Stream Graph → Window Scaling**:

- La ventana TCP crece progresivamente al inicio (slow start).
- Si hay congestión, la ventana se reduce bruscamente (visible como dientes de sierra).
- Las retransmisiones aparecen coloreadas en **negro** en la lista de paquetes.

### 10.6 Análisis de iPerf3 en modo UDP

**Filtro en Wireshark:**
```
udp.port == 5201
```

Observar:
- La tasa de llegada de paquetes en **Statistics → IO Graph** (eje Y en bits/s).
- Gaps en la secuencia de datagramas iPerf3 indican descarte por congestión.

### 10.7 Gráfico IO — comparar tráfico RTP vs iPerf3

1. Ir a **Statistics → I/O Graphs**
2. Agregar dos series:
   - **Serie 1:** Filtro `rtp` — color azul — muestra tráfico de video
   - **Serie 2:** Filtro `udp.port == 5201` — color rojo — muestra iPerf3 UDP
3. Unidad: **Bits/s**

Este gráfico permite ver visualmente cómo iPerf3 a 50 Mbps aplasta el tráfico RTP, y cómo al aplicar QoS el video recupera su ancho de banda.

> **Captura de pantalla recomendada:** IO Graph con ambas series activas durante la prueba UDP 50 Mbps.

### 10.8 Guardar la captura

```bash
# Desde terminal en el Monitor, guardar la sesión completa
tshark -i enp0s8 -f "host 192.168.1.10" -w /tmp/captura_p2_completa.pcap

# O desde la GUI: File → Save As → captura_p2.pcapng
```

---

## 11. Paso 9 — Limitación de ancho de banda en el Router (QoS)

Configurar el router c3725 para limitar el tráfico de video a 10 Mbps.

### 11.1 Configurar la política en el c3725

```cisco
enable
configure terminal

! === ACL que identifica tráfico de video ===
ip access-list extended ACL_VIDEO
 permit tcp any any eq 554
 permit tcp any eq 554 any
 permit udp any 192.168.1.0 0.0.0.255 range 1024 65535
exit

! === Clase de tráfico ===
class-map match-any CM_VIDEO
 match access-group name ACL_VIDEO
exit

! === Política: 10 Mbps, descartar exceso ===
policy-map PM_LIMITE_VIDEO
 class CM_VIDEO
  police rate 10000000 bps
   conform-action transmit
   exceed-action drop
 class class-default
  fair-queue
exit

! === Aplicar en la interfaz LAN ===
interface FastEthernet0/0
 service-policy input PM_LIMITE_VIDEO
exit

end
write memory
```

### 11.2 Verificación en el router

```cisco
show policy-map interface FastEthernet0/0
show class-map
show access-lists ACL_VIDEO
```

### 11.3 Verificar el efecto en Wireshark

Con la política activa, repetir la prueba iPerf3 UDP 50 Mbps y observar en el **IO Graph** de Wireshark (VM Monitor):

- La curva roja de iPerf3 queda limitada y no supera ~10 Mbps
- La curva azul de RTP se mantiene estable
- En el reporte de iPerf3 el campo `lost/total datagrams` sube notablemente

---

## 12. Paso 10 — Integración con NetFlow/sFlow (conexión con Punto 1)

### 12.1 Instalar softflowd en la VM Servidor

```bash
sudo apt install -y softflowd
```

### 12.2 Exportar NetFlow al colector del Punto 1

```bash
# Reemplazar <IP_Admin_VM_P1> por la IP real del Admin VM del Punto 1 (enp0s3)
sudo softflowd -i enp0s8 -v 9 \
  -n <IP_Admin_VM_P1>:2055 \
  -t maxlife=60 -b 2048 &
```

### 12.3 Verificar en el Admin VM del Punto 1

```bash
sudo mysql -u pmacct -ppmacctpass pmacct_db \
  -e "SELECT src_host, dst_host, proto, bytes \
      FROM flows WHERE src_host='192.168.1.10' \
      OR dst_host='192.168.1.10' \
      ORDER BY stamp_inserted DESC LIMIT 10;" --table
```

Los flujos RTSP e iPerf3 del Punto 2 aparecerán en el dashboard de Grafana del Punto 1.

---

## 13. Respuestas a preguntas del laboratorio

### Pregunta 1: Sesión RTSP completa — ¿qué mensajes se intercambian?

Al capturar con el filtro `rtsp` en Wireshark, se observa el siguiente intercambio completo entre el Cliente Monitor y el Servidor:

**DESCRIBE** — El cliente solicita la descripción del stream. El servidor responde con un documento SDP que especifica el codec de video (H.264), la frecuencia de muestreo, y los puertos RTP/RTCP que se usarán. Este paquete es visible en el panel inferior de Wireshark expandiendo la sección Session Description Protocol.

**SETUP** — El cliente negocia los puertos UDP locales donde recibirá los paquetes RTP. El servidor confirma los puertos asignados en la respuesta `200 OK`.

**PLAY** — El cliente indica al servidor que comience a enviar el stream. A partir de este mensaje Wireshark muestra la avalancha de paquetes RTP en puertos UDP altos, visible claramente en el IO Graph como un aumento súbito en la tasa de bits.

**TEARDOWN** — Al cerrar VLC, el cliente envía este mensaje y el servidor libera los recursos del stream.

---

### Pregunta 2: ¿Cómo se relacionan los números de secuencia RTP con los fotogramas YOLO?

En Wireshark, con el filtro `rtp` activo, cada paquete muestra en su cabecera el campo **Sequence number** (incrementa en 1 por paquete) y el **Timestamp** (refleja el tiempo de captura del fotograma). Un fotograma de video H.264 puede fragmentarse en múltiples paquetes RTP consecutivos con el mismo timestamp y sequence numbers consecutivos.

YOLO procesa un fotograma completo cuando el decoder de video ha recibido todos sus fragmentos RTP. Si en la ventana de Wireshark se observa un gap en los sequence numbers (por ejemplo, salta de 1450 a 1453), significa que se perdieron dos fragmentos de un fotograma. El decoder descarta ese fotograma incompleto y YOLO no puede procesarlo, lo que se refleja como una caída en el FPS visible en la consola del Servidor. La herramienta **Telephony → RTP → RTP Streams → Analyze** de Wireshark muestra el gráfico de paquetes perdidos en el tiempo, correlacionable directamente con las caídas de FPS.

---

### Pregunta 3: Impacto del modelo YOLO en el tráfico capturado por Wireshark

El procesamiento YOLO es local y no genera tráfico de red adicional por sí mismo. Sin embargo, al cambiar de `yolov8n.pt` a `yolov8l.pt`, la CPU de la VM Servidor se satura con las inferencias más pesadas. ffmpeg, que corre en el mismo sistema, pierde tiempo de CPU y comienza a generar frames con mayor latencia.

En Wireshark esto es observable en la herramienta **RTP Stream Analysis**: el jitter entre paquetes RTP aumenta porque ffmpeg no entrega los frames al codificador a intervalos regulares. También se reduce la tasa de bits total del stream, visible en el IO Graph. Con el modelo nano (`yolov8n.pt`), el IO Graph muestra una línea estable y el jitter en RTP Stream Analysis es mínimo.

---

### Pregunta 4: Impacto de iPerf3 UDP 50 Mbps en el FPS de YOLO

Al lanzar `iperf3 -u -b 50M` desde el Cliente Tráfico, el IO Graph de Wireshark en el Monitor muestra cómo la curva roja (iPerf3 UDP) satura la red y aplasta la curva azul (RTP). El switch IOU L2 empieza a descartar paquetes cuando sus buffers se llenan. Los fragmentos RTP perdidos producen fotogramas incompletos que el decoder descarta, y el FPS de YOLO cae de forma visible en la consola del Servidor.

Con la política QoS activa en el c3725 (10 Mbps para video), el IO Graph muestra cómo la curva azul del RTP se mantiene estable aunque la curva roja de iPerf3 intente superar el límite. El FPS de YOLO se recupera y el reporte de iPerf3 muestra un aumento en `lost/total datagrams` porque ahora es iPerf3 el que absorbe el descarte del router.

---

### Pregunta 5: ¿Qué se vería primero — caída en FPS o aumento en pérdida iPerf3?

**Se vería primero el aumento en la pérdida de paquetes reportada por iPerf3.**

iPerf3 UDP mide la pérdida directamente en cada intervalo de reporte (cada segundo con `-i 1`). En cuanto el switch o router descarta paquetes por congestión, iPerf3 lo detecta de inmediato comparando los datagrams enviados contra los recibidos. Este resultado aparece en el terminal del Cliente Tráfico prácticamente en tiempo real.

La caída en el FPS de YOLO tiene mayor latencia de manifestación: requiere que se pierdan fragmentos RTP, que el decoder acumule suficientes fotogramas incompletos, y que el bucle de inferencia de YOLO detecte la reducción de frames válidos disponibles. Los decoders de video tienen buffers internos y mecanismos de ocultamiento de errores (error concealment) que retrasan la manifestación visible de la degradación varios segundos después de iniciada la congestión.

En Wireshark esta secuencia también es observable: los gaps en los sequence numbers RTP aparecen en la lista de paquetes casi simultáneamente con los gaps en los datagramas iPerf3, pero el efecto en el FPS de YOLO llega segundos después.

---

### Pregunta 6: Comandos para limitar tráfico de video a 10 Mbps en router Cisco c3725

```cisco
ip access-list extended ACL_VIDEO
 permit tcp any any eq 554
 permit tcp any eq 554 any
 permit udp any 192.168.1.0 0.0.0.255 range 1024 65535
exit

class-map match-any CM_VIDEO
 match access-group name ACL_VIDEO
exit

policy-map PM_LIMITE_VIDEO
 class CM_VIDEO
  police rate 10000000 bps
   conform-action transmit
   exceed-action drop
 class class-default
  fair-queue
exit

interface FastEthernet0/0
 service-policy input PM_LIMITE_VIDEO
exit
```

---

### Pregunta 7: Métricas en Wireshark que confirman la limitación de ancho de banda

**IO Graph (Statistics → I/O Graphs):** Con la serie filtrada por `rtp` en azul y `udp.port==5201` en rojo, la curva roja no supera ~10 Mbps aunque iPerf3 envíe 50 Mbps. Esta es la evidencia visual más directa de que la política está activa.

**Pérdida de paquetes iPerf3:** En la lista de paquetes de Wireshark con filtro `udp.port==5201`, los números de secuencia de los datagramas tienen saltos, confirmando que el router descarta los paquetes que exceden el límite configurado.

**RTP Stream Analysis (Telephony → RTP → RTP Streams):** El jitter del stream de video disminuye al aplicar QoS comparado con la prueba sin política, porque el tráfico RTSP/RTP ya no compite en igualdad de condiciones con iPerf3 por el ancho de banda del router.

**Ventana TCP estabilizada:** En el gráfico Window Scaling de las conexiones TCP (Statistics → TCP Stream Graph), la ventana se estabiliza en valores más conservadores en presencia de la política, indicando que el mecanismo de control de congestión TCP reacciona a los descartes del router reduciendo su tasa de envío.

---

## Referencias

- Ultralytics YOLOv8 Documentation — docs.ultralytics.com
- ffmpeg RTSP Streaming Guide — ffmpeg.org/documentation.html
- Wireshark User Guide — wireshark.org/docs/wsug_html_chunked
- Wireshark RTP Analysis — wiki.wireshark.org/RTP
- Cisco IOS QoS Configuration Guide — cisco.com/c/en/us/td/docs/ios/qos
- iPerf3 Documentation — iperf.fr
- Repositorio del docente — github.com/dialejobv/conmutacion_teletrafico
- RFC 3550 — RTP: A Transport Protocol for Real-Time Applications
- RFC 2326 — Real Time Streaming Protocol (RTSP)
- VLC Documentation — videolan.org/vlc
