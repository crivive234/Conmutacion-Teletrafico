# Laboratorio 2 — Punto 1: Monitoreo con Grafana, Zabbix y pmacct
**Asignatura:** Conmutación y Teletráfico  
**Docente:** Diego Alejandro Barragán Vargas — Fundación Universitaria Compensar

---

## Tabla de Contenidos
1. [Resumen de infraestructura real](#1-resumen-de-infraestructura-real)
2. [Topología en GNS3](#2-topología-en-gns3)
3. [Paso 1 — Configuración del Switch IOU L2 en GNS3](#3-paso-1--configuración-del-switch-iou-l2-en-gns3)
4. [Paso 2 — Redes Docker Desktop en Windows](#4-paso-2--redes-docker-desktop-en-windows)
5. [Paso 3 — Admin VM Debian](#5-paso-3--admin-vm-debian)
6. [Paso 4 — Nodos SubRed 1](#6-paso-4--nodos-subred-1)
7. [Paso 5 — Nodos SubRed 2](#7-paso-5--nodos-subred-2)
8. [Paso 6 — pmacct: NetFlow + sFlow + IPFIX](#8-paso-6--pmacct-netflow--sflow--ipfix)
9. [Paso 7 — Zabbix](#9-paso-7--zabbix)
10. [Paso 8 — Grafana y Dashboard](#10-paso-8--grafana-y-dashboard)
11. [Paso 9 — Generación de tráfico con iPerf3](#11-paso-9--generación-de-tráfico-con-iperf3)
12. [Paso 10 — Captura con Wireshark](#12-paso-10--captura-con-wireshark)
13. [Respuestas a preguntas del laboratorio](#13-respuestas-a-preguntas-del-laboratorio)

---

## 1. Resumen de infraestructura real

### Adaptadores Host-Only de VirtualBox

| Adaptador Windows | Host-Only VirtualBox | IP del Host Windows | Uso en el laboratorio |
|---|---|---|---|
| Ethernet 4 | Adapter #1 | 192.168.56.1/24 | Cloud-SR1 → SubRed 1 |
| Ethernet 5 | Adapter #2 | 192.168.147.1/24 | Cloud-SR2 → SubRed 2 |
| Ethernet 6 | Adapter #3 | 192.168.74.1/24 | Cloud-Admin → trunk Admin VM |
| Ethernet 7 | Adapter #4 | 192.168.252.1/24 | GNS3 VM (exclusivo) |
| Ethernet 8 | Adapter #5 | 192.168.167.1/24 | SPAN → eth2 Admin VM (sin IP en VM) |

### Tabla de IPs del laboratorio

| Nodo | Tipo | SubRed | VLAN | IP | Gateway |
|---|---|---|---|---|---|
| Admin VM eth1.10 | Subinterfaz | 1 | 10 | 192.168.56.1/24 | — |
| Admin VM eth1.20 | Subinterfaz | 2 | 20 | 192.168.147.1/24 | — |
| Admin VM eth2 | SPAN | — | — | Sin IP (promiscuo) | — |
| GNS3 VM | VM Ubuntu | — | — | 192.168.252.100 | 192.168.252.1 |
| Arch Linux VM | VM VirtualBox | 1 | 10 | 192.168.56.10/24 | 192.168.56.1 |
| Rocky Linux VM | VM VirtualBox | 1 | 10 | 192.168.56.20/24 | 192.168.56.1 |
| Fedora Container | Docker Desktop | 1 | 10 | 192.168.56.30/24 | 192.168.56.1 |
| Ubuntu VM | VM VirtualBox | 2 | 20 | 192.168.147.10/24 | 192.168.147.1 |
| Alpine Container | Docker Desktop | 2 | 20 | 192.168.147.20/24 | 192.168.147.1 |
| Kali Container | Docker Desktop | 2 | 20 | 192.168.147.30/24 | 192.168.147.1 |

### Puertos del Switch IOU L2 (i86bi-linux-l2)

| Puerto Switch | Modo | Conectado a | Propósito |
|---|---|---|---|
| e3/2 | Trunk (VLAN 10,20) | Cloud-Admin (Ethernet 6) | Admin VM eth1 |
| e0/1 | Access VLAN 10 | Cloud-SR1 (Ethernet 4) | SubRed 1 |
| e0/2 | Access VLAN 20 | Cloud-SR2 (Ethernet 5) | SubRed 2 |
| e0/3 | SPAN destino | Cloud-SPAN (Ethernet 8) | Admin VM eth2 captura |

> **Nota:** El switch IOU L2 usa notación `eX/Y` (Ethernet) en lugar de `gi0/X` (GigabitEthernet) de los switches físicos Cisco.

---

## 2. Topología en GNS3

```
                    Admin VM — Debian (VirtualBox)
                   eth1 → Ethernet 6 (192.168.74.x)
                   eth2 → Ethernet 8 (sin IP, SPAN)
                          │                │
                    Cloud-Admin        Cloud-SPAN
                    Ethernet 6         Ethernet 8
                          │                │
                        e3/2            e0/3
                          │                │
                       SW2960 IOU L2
                       (GNS3 VM)
                          │         │
                        e0/1      e0/2
                          │         │
                     Cloud-SR1  Cloud-SR2
                     Ethernet4  Ethernet5
                   (192.168.56) (192.168.147)
                          │         │
              ┌───────────┤     ┌───┤
         Arch Linux   Rocky   Ubuntu  Alpine  Kali
         VM           Linux   VM      Cont.   Cont.
         .56.10       VM      .147.10 .147.20 .147.30
                      .56.20
              Fedora Container
              .56.30
         (VirtualBox / Docker Desktop en Windows)
```

### Nodos en GNS3 (solo estos 5)

| Nodo GNS3 | Tipo | Interfaz asignada |
|---|---|---|
| SW2960 | Switch IOU L2 | — |
| Cloud-Admin | Cloud node | Ethernet 6 |
| Cloud-SR1 | Cloud node | Ethernet 4 |
| Cloud-SR2 | Cloud node | Ethernet 5 |
| Cloud-SPAN | Cloud node | Ethernet 8 |

Las VMs y contenedores **no aparecen en GNS3** — corren en VirtualBox y Docker Desktop de Windows y se conectan a través de los adaptadores Host-Only.

---

## 3. Paso 1 — Configuración del Switch IOU L2 en GNS3

Iniciar el proyecto en GNS3 con la GNS3 VM corriendo. Hacer clic derecho sobre el switch → Console.

```cisco
! === PASO 1: Crear VLANs ===
enable
configure terminal

vlan 10
 name SUBRED_1
vlan 20
 name SUBRED_2
exit

! === PASO 2: Puerto trunk hacia Admin VM (e3/2) ===
interface ethernet 3/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 no shutdown
exit

! === PASO 3: Puerto acceso SubRed 1 (e0/1) ===
interface ethernet 0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

! === PASO 4: Puerto acceso SubRed 2 (e0/2) ===
interface ethernet 0/2
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit

! === PASO 5: SPAN — espejo de ambas VLANs hacia e0/3 ===
monitor session 1 source vlan 10,20 both
monitor session 1 destination interface ethernet 0/3

! === PASO 6: IP de gestión del switch + SNMP ===
interface vlan 1
 ip address 192.168.56.100 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.56.1

snmp-server community public RO
snmp-server community private RW
snmp-server location "Laboratorio_Compensar"
snmp-server contact "admin@lab.com"
snmp-server host 192.168.74.X public

! === PASO 7: Guardar ===
end
write memory
```

### Verificación del switch

```cisco
show vlan brief
show interfaces trunk
show monitor session 1
show running-config
```

---

## 4. Paso 2 — Redes Docker Desktop en Windows

Abrir **PowerShell como Administrador**.

```powershell
# Red Docker para SubRed 1 — mismo rango que Host-Only Adapter #1
docker network create `
  --driver bridge `
  --subnet 192.168.56.0/24 `
  --gateway 192.168.56.1 `
  --opt "com.docker.network.bridge.name"="br-vlan10" `
  vlan10_net

# Red Docker para SubRed 2 — mismo rango que Host-Only Adapter #2
docker network create `
  --driver bridge `
  --subnet 192.168.147.0/24 `
  --gateway 192.168.147.1 `
  --opt "com.docker.network.bridge.name"="br-vlan20" `
  vlan20_net

# Verificar redes creadas
docker network ls
docker network inspect vlan10_net
docker network inspect vlan20_net
```

> **Nota importante:** Docker Desktop en Windows crea un adaptador virtual para cada red bridge. El rango 192.168.56.0/24 coincide con el Host-Only Adapter #1, por lo que Windows puede enrutar tráfico entre contenedores Docker y las VMs de VirtualBox en la misma subred. Si Docker muestra conflicto de subred, verificar en Docker Desktop → Settings → Resources → Network que no haya rangos solapados predefinidos.

---

## 5. Paso 3 — Admin VM Debian

La VM Debian en VirtualBox debe tener **tres adaptadores** configurados:
- **Adaptador 1:** NAT → `eth0` (internet)
- **Adaptador 2:** Host-Only Adapter #3 → `eth1` (trunk al switch e3/2)
- **Adaptador 3:** Host-Only Adapter #5 → `eth2` (SPAN, sin IP)

### 5.1 Instalación de paquetes

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  vlan wireshark tshark iperf3 \
  zabbix-server-mysql zabbix-frontend-php \
  zabbix-apache-conf zabbix-sql-scripts zabbix-agent \
  mariadb-server \
  pmacct softflowd \
  snmp snmpd snmp-mibs-downloader \
  curl gnupg apt-transport-https \
  net-tools tcpdump
```

### 5.2 Configuración trunk 802.1q y subinterfaces VLAN

```bash
# Cargar módulo 802.1q
sudo modprobe 8021q
echo "8021q" | sudo tee -a /etc/modules

# Verificar nombre real de la interfaz
ip link show
# Buscar la interfaz conectada al Adapter #3
# Puede llamarse eth1, enp0s8, etc. Ajustar según corresponda
```

Editar `/etc/network/interfaces`:

```bash
sudo nano /etc/network/interfaces
```

Agregar al final:

```
# Interfaz física trunk (conectada a Host-Only Adapter #3)
auto eth1
iface eth1 inet manual
    up ip link set $IFACE up

# Subinterfaz VLAN 10 — Gateway SubRed 1
auto eth1.10
iface eth1.10 inet static
    address 192.168.56.1
    netmask 255.255.255.0
    vlan-raw-device eth1

# Subinterfaz VLAN 20 — Gateway SubRed 2
auto eth1.20
iface eth1.20 inet static
    address 192.168.147.1
    netmask 255.255.255.0
    vlan-raw-device eth1
```

```bash
# Aplicar configuración
sudo ifup eth1
sudo ifup eth1.10 eth1.20

# Verificar
ip addr show eth1.10
ip addr show eth1.20
```

### 5.3 Habilitar IP forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Verificar
sysctl net.ipv4.ip_forward
```

### 5.4 Configurar interfaz SPAN eth2 (sin IP, modo promiscuo)

```bash
# eth2 está conectada a Host-Only Adapter #5
sudo ip link set eth2 up
sudo ip addr flush dev eth2
sudo ip link set eth2 promisc on

# Persistir en /etc/network/interfaces
sudo tee -a /etc/network/interfaces <<EOF

# Interfaz SPAN — captura pasiva sin IP
auto eth2
iface eth2 inet manual
    up ip link set \$IFACE up promisc on
    down ip link set \$IFACE down
EOF
```

### 5.5 Verificar conectividad

```bash
# Desde Admin VM, hacer ping a los gateways de cada subred
ping -c 3 192.168.56.1    # propio gateway SR1
ping -c 3 192.168.147.1   # propio gateway SR2

# Una vez que los nodos estén configurados
ping -c 3 192.168.56.10   # Arch Linux
ping -c 3 192.168.147.10  # Ubuntu
```

---

## 6. Paso 4 — Nodos SubRed 1

Todos los nodos de SubRed 1 deben tener:
- IP en rango 192.168.56.0/24
- Gateway: 192.168.56.1
- En VirtualBox: Adaptador Host-Only Adapter #1 (Ethernet 4)

### 6.1 Arch Linux VM (192.168.56.10)

Adaptadores VirtualBox:
- Adaptador 1: NAT → eth0
- Adaptador 2: Host-Only Adapter #1 → eth1

```bash
# Configurar IP estática con systemd-networkd
sudo tee /etc/systemd/network/20-eth1.network <<EOF
[Match]
Name=eth1

[Network]
Address=192.168.56.10/24
Gateway=192.168.56.1
DNS=8.8.8.8
EOF

sudo systemctl enable --now systemd-networkd

# Verificar
ip addr show eth1
ping -c 3 192.168.56.1

# Instalar Zabbix Agent e iPerf3
sudo pacman -Sy zabbix-agent iperf3

# Configurar Zabbix Agent
sudo sed -i 's/^Server=.*/Server=192.168.56.1/' /etc/zabbix/zabbix_agentd.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=192.168.56.1/' /etc/zabbix/zabbix_agentd.conf
sudo sed -i 's/^Hostname=.*/Hostname=Arch-Linux-VM/' /etc/zabbix/zabbix_agentd.conf

sudo systemctl enable --now zabbix-agentd
```

### 6.2 Rocky Linux VM (192.168.56.20)

Adaptadores VirtualBox: igual que Arch Linux.

```bash
# Configurar IP estática
sudo nmcli con mod eth1 ipv4.addresses 192.168.56.20/24
sudo nmcli con mod eth1 ipv4.gateway 192.168.56.1
sudo nmcli con mod eth1 ipv4.dns 8.8.8.8
sudo nmcli con mod eth1 ipv4.method manual
sudo nmcli con up eth1

# Instalar Zabbix Agent
sudo rpm -Uvh https://repo.zabbix.com/zabbix/6.4/rhel/9/x86_64/zabbix-release-6.4-1.el9.noarch.rpm
sudo dnf install -y zabbix-agent iperf3

sudo sed -i 's/^Server=.*/Server=192.168.56.1/' /etc/zabbix/zabbix_agentd.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=192.168.56.1/' /etc/zabbix/zabbix_agentd.conf
sudo sed -i 's/^Hostname=.*/Hostname=Rocky-Linux-VM/' /etc/zabbix/zabbix_agentd.conf

sudo systemctl enable --now zabbix-agent

# Verificar
ping -c 3 192.168.56.1
```

### 6.3 Contenedor Fedora (192.168.56.30 — Docker Desktop Windows)

```powershell
# PowerShell — levantar contenedor Fedora en SubRed 1
docker run -dit `
  --name fedora-subred1 `
  --network vlan10_net `
  --ip 192.168.56.30 `
  --hostname fedora-container `
  fedora:latest

# Verificar que esté corriendo
docker ps

# Entrar al contenedor
docker exec -it fedora-subred1 bash
```

```bash
# Dentro del contenedor Fedora
dnf install -y zabbix-agent iperf3 iproute iputils net-tools

# Configurar Zabbix Agent
cat > /etc/zabbix/zabbix_agentd.conf <<EOF
Server=192.168.56.1
ServerActive=192.168.56.1
Hostname=Fedora-Container
EOF

zabbix_agentd

# Verificar conectividad
ping -c 3 192.168.56.1
```

---

## 7. Paso 5 — Nodos SubRed 2

Todos los nodos de SubRed 2 deben tener:
- IP en rango 192.168.147.0/24
- Gateway: 192.168.147.1
- En VirtualBox: Adaptador Host-Only Adapter #2 (Ethernet 5)

### 7.1 Ubuntu VM (192.168.147.10)

Adaptadores VirtualBox:
- Adaptador 1: NAT → eth0
- Adaptador 2: Host-Only Adapter #2 → eth1

```bash
# Configurar IP estática con netplan
sudo tee /etc/netplan/01-eth1.yaml <<EOF
network:
  version: 2
  ethernets:
    eth1:
      addresses:
        - 192.168.147.10/24
      routes:
        - to: default
          via: 192.168.147.1
      nameservers:
        addresses: [8.8.8.8]
EOF

sudo netplan apply

# Instalar Zabbix Agent e iPerf3
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo apt update
sudo apt install -y zabbix-agent iperf3

sudo sed -i 's/^Server=.*/Server=192.168.147.1/' /etc/zabbix/zabbix_agentd.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=192.168.147.1/' /etc/zabbix/zabbix_agentd.conf
sudo sed -i 's/^Hostname=.*/Hostname=Ubuntu-VM/' /etc/zabbix/zabbix_agentd.conf

sudo systemctl enable --now zabbix-agent

# Verificar
ping -c 3 192.168.147.1
```

### 7.2 Contenedor Alpine (192.168.147.20 — Docker Desktop Windows)

```powershell
docker run -dit `
  --name alpine-subred2 `
  --network vlan20_net `
  --ip 192.168.147.20 `
  --hostname alpine-container `
  alpine:latest

docker exec -it alpine-subred2 sh
```

```sh
# Dentro del contenedor Alpine
apk update
apk add zabbix-agent iperf3 iproute2 iputils

cat > /etc/zabbix/zabbix_agentd.conf <<EOF
Server=192.168.147.1
ServerActive=192.168.147.1
Hostname=Alpine-Container
EOF

zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf

# Verificar
ping -c 3 192.168.147.1
```

### 7.3 Contenedor Kali Linux (192.168.147.30 — Docker Desktop Windows)

```powershell
docker run -dit `
  --name kali-subred2 `
  --network vlan20_net `
  --ip 192.168.147.30 `
  --hostname kali-container `
  kalilinux/kali-rolling:latest

docker exec -it kali-subred2 bash
```

```bash
# Dentro del contenedor Kali
apt update
apt install -y zabbix-agent iperf3 net-tools iproute2

cat > /etc/zabbix/zabbix_agentd.conf <<EOF
Server=192.168.147.1
ServerActive=192.168.147.1
Hostname=Kali-Container
EOF

zabbix_agentd

# Verificar
ping -c 3 192.168.147.1
```

---

## 8. Paso 6 — pmacct (NetFlow + sFlow + IPFIX)

Todo se configura en la **Admin VM Debian**.

### 8.1 Base de datos MySQL para pmacct

```bash
sudo mysql -u root <<'SQL'
CREATE DATABASE pmacct_db CHARACTER SET utf8mb4;
CREATE USER 'pmacct'@'localhost' IDENTIFIED BY 'pmacctpass';
GRANT ALL PRIVILEGES ON pmacct_db.* TO 'pmacct'@'localhost';
FLUSH PRIVILEGES;
EXIT;
SQL

sudo mysql -u pmacct -ppmacctpass pmacct_db <<'SQL'
CREATE TABLE flows (
    stamp_inserted DATETIME,
    src_host       CHAR(15),
    dst_host       CHAR(15),
    proto          TINYINT,
    src_port       INT,
    dst_port       INT,
    tos            TINYINT,
    packets        BIGINT,
    bytes          BIGINT
);
SQL
```

### 8.2 Opción 1 — NetFlow v9 con softflowd (desde SPAN eth2)

```bash
# softflowd lee el tráfico espejado en eth2 y genera NetFlow v9
sudo softflowd -i eth2 -v 9 -n 127.0.0.1:2055 -t maxlife=60 -b 2048

# Servicio systemd para arranque automático
sudo tee /etc/systemd/system/softflowd.service <<EOF
[Unit]
Description=softflowd NetFlow v9 generator
After=network.target

[Service]
ExecStart=/usr/sbin/softflowd -i eth2 -v 9 -n 127.0.0.1:2055 -t maxlife=60 -b 2048
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now softflowd
```

### 8.3 Opción 2 — sFlow v5 con pmacct como probe

```bash
sudo mkdir -p /etc/pmacct

sudo tee /etc/pmacct/sflow_probe.conf <<EOF
!
daemonize: true
pidfile: /var/run/pmacct_sflow.pid
interface: eth2
plugins: sflow
sflow_target: 127.0.0.1:6343
sflow_version: 5
sflow_sampling_rate: 1
!
EOF

sudo pmacctd -f /etc/pmacct/sflow_probe.conf
```

### 8.4 Opción 3 — IPFIX con pmacct como probe

```bash
sudo tee /etc/pmacct/ipfix_probe.conf <<EOF
!
daemonize: true
pidfile: /var/run/pmacct_ipfix.pid
interface: eth2
plugins: nfprobe
nfprobe_version: 10
nfprobe_receiver: 127.0.0.1:4739
!
EOF

sudo pmacctd -f /etc/pmacct/ipfix_probe.conf
```

### 8.5 Colector central pmacct (recibe los tres protocolos)

```bash
sudo tee /etc/pmacct/collector.conf <<EOF
!
daemonize: true
pidfile: /var/run/pmacct_collector.pid
plugins: mysql
aggregate: src_host, dst_host, proto, src_port, dst_port, tos
sql_table: flows
sql_db: pmacct_db
sql_user: pmacct
sql_passwd: pmacctpass
sql_history: 1m
sql_refresh_time: 60
nf_ports: 2055
sflow_ports: 6343
ipfix_ports: 4739
!
EOF

sudo pmacctd -f /etc/pmacct/collector.conf

# Verificar que está recibiendo flujos (después de generar tráfico)
sudo mysql -u pmacct -ppmacctpass pmacct_db \
  -e "SELECT * FROM flows ORDER BY stamp_inserted DESC LIMIT 10;" --table
```

---

## 9. Paso 7 — Zabbix

```bash
# Instalar repositorio Zabbix 6.4 para Debian 12
wget https://repo.zabbix.com/zabbix/6.4/debian/pool/main/z/zabbix-release/zabbix-release_6.4-1+debian12_all.deb
sudo dpkg -i zabbix-release_6.4-1+debian12_all.deb
sudo apt update

sudo apt install -y \
  zabbix-server-mysql zabbix-frontend-php \
  zabbix-apache-conf zabbix-sql-scripts \
  zabbix-agent mariadb-server

# Crear base de datos Zabbix
sudo mysql -u root <<'SQL'
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'zabbixpass';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;
SQL

# Importar esquema
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
  sudo mysql -uzabbix -pzabbixpass zabbix

# Configurar contraseña en zabbix_server.conf
sudo sed -i 's/# DBPassword=/DBPassword=zabbixpass/' \
  /etc/zabbix/zabbix_server.conf

# Iniciar servicios
sudo systemctl enable --now zabbix-server zabbix-agent apache2

# Verificar
sudo systemctl status zabbix-server
```

### Configuración web

1. Abrir: `http://<IP_eth0_Admin>/zabbix`
2. Login: `Admin` / `zabbix`
3. Agregar hosts en **Configuration → Hosts → Create Host**:

| Host name | IP | Template |
|---|---|---|
| Arch-Linux-VM | 192.168.56.10 | Linux by Zabbix agent |
| Rocky-Linux-VM | 192.168.56.20 | Linux by Zabbix agent |
| Fedora-Container | 192.168.56.30 | Linux by Zabbix agent |
| Ubuntu-VM | 192.168.147.10 | Linux by Zabbix agent |
| Alpine-Container | 192.168.147.20 | Linux by Zabbix agent |
| Kali-Container | 192.168.147.30 | Linux by Zabbix agent |
| SW2960-GNS3 | 192.168.56.100 | Network generic device by SNMP |

---

## 10. Paso 8 — Grafana y Dashboard

```bash
# Instalar Grafana en Admin VM
sudo apt install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt update && sudo apt install -y grafana

sudo systemctl enable --now grafana-server

# Verificar
sudo ss -tlnp | grep 3000
```

### Acceso y fuentes de datos

1. Abrir: `http://<IP_eth0_Admin>:3000`
2. Login: `admin` / `admin`

**Instalar plugin Zabbix:**
```bash
sudo grafana-cli plugins install alexanderzobnin-zabbix-app
sudo systemctl restart grafana-server
```

**Fuente de datos MySQL (pmacct):**
- Host: `localhost:3306`
- Database: `pmacct_db`
- User: `pmacct` | Password: `pmacctpass`

**Fuente de datos Zabbix:**
- URL: `http://localhost/zabbix/api_jsonrpc.php`
- User: `Admin` | Password: `zabbix`

### Paneles del Dashboard

**Panel 1 — Tráfico por segundo:**
```sql
SELECT
  UNIX_TIMESTAMP(stamp_inserted) AS time,
  SUM(bytes) / 60 AS bytes_per_second,
  SUM(packets) / 60 AS packets_per_second
FROM flows
WHERE $__timeFilter(stamp_inserted)
GROUP BY stamp_inserted
ORDER BY stamp_inserted ASC
```

**Panel 2 — Top 5 conversaciones:**
```sql
SELECT
  src_host,
  dst_host,
  SUM(bytes) AS total_bytes,
  SUM(packets) AS total_packets
FROM flows
WHERE $__timeFilter(stamp_inserted)
GROUP BY src_host, dst_host
ORDER BY total_bytes DESC
LIMIT 5
```

**Panel 3 — CPU hosts (Zabbix):**
- Data source: Zabbix
- Host: todos los nodos
- Item: CPU utilization
- Tipo: Time series

**Panel 4 — Tráfico de red (Zabbix):**
- Item: Network interface Bits received/sent
- Tipo: Time series

**Panel 5 — Alertas activas (Zabbix):**
- Tipo: Zabbix Problems

---

## 11. Paso 9 — Generación de tráfico con iPerf3

### Prueba TCP — Arch Linux → Ubuntu

```bash
# En Ubuntu VM (servidor)
iperf3 -s -p 5201

# En Arch Linux VM (cliente) — TCP 60s, 4 flujos paralelos
iperf3 -c 192.168.147.10 -p 5201 -t 60 -i 5 -P 4
```

### Prueba UDP — Rocky Linux → Alpine (simulando streaming)

```bash
# En Alpine Container
iperf3 -s -p 5202

# En Rocky Linux VM — UDP 10 Mbps
iperf3 -c 192.168.147.20 -p 5202 -u -b 10M -t 60 -i 1
```

### Verificar flujos en tiempo real

```bash
# En Admin VM — ver flujos capturados por pmacct
sudo mysql -u pmacct -ppmacctpass pmacct_db \
  -e "SELECT stamp_inserted, src_host, dst_host, proto, bytes \
      FROM flows ORDER BY stamp_inserted DESC LIMIT 20;" --table
```

---

## 12. Paso 10 — Captura con Wireshark

```bash
# Instalar en Admin VM
sudo apt install -y wireshark tshark
sudo dpkg-reconfigure wireshark-common   # Seleccionar Sí
sudo usermod -aG wireshark $USER
newgrp wireshark

# Capturar en eth2 (interfaz SPAN — ve todo el tráfico del switch)
sudo tshark -i eth2 \
  -f "host 192.168.56.10 or host 192.168.147.10" \
  -w /tmp/trafico_lab.pcap

# Ver estadísticas por segundo durante captura
sudo tshark -i eth2 -qz io,stat,1 -a duration:65

# Analizar protocolos de flujo en loopback
sudo tshark -i lo -f "udp port 2055" -V | head -30   # NetFlow
sudo tshark -i lo -f "udp port 6343" -V | head -30   # sFlow
sudo tshark -i lo -f "udp port 4739" -V | head -30   # IPFIX
```

---

## 13. Respuestas a preguntas del laboratorio

### Pregunta 1: Describir paso a paso el desarrollo de la arquitectura

La arquitectura se construyó en seis capas sobre infraestructura híbrida (GNS3 + VirtualBox + Docker Desktop en Windows 11):

**Capa 1 — Conmutación en GNS3:** Se emula un switch Cisco IOU L2 dentro de la GNS3 VM corriendo en VirtualBox. El switch tiene tres Cloud nodes que actúan como puentes hacia los adaptadores Host-Only de Windows: Cloud-Admin (Ethernet 6) conectado al puerto trunk e3/2, Cloud-SR1 (Ethernet 4) al puerto de acceso VLAN 10 e0/1, Cloud-SR2 (Ethernet 5) al puerto de acceso VLAN 20 e0/2, y Cloud-SPAN (Ethernet 8) al puerto espejo e0/3.

**Capa 2 — Enrutamiento inter-VLAN (router-on-a-stick):** La VM Admin Debian actúa como router entre subredes. Su interfaz eth1 se conecta al trunk del switch a través del adaptador Host-Only #3. Se crean dos subinterfaces VLAN: eth1.10 con IP 192.168.56.1 (gateway SubRed 1) y eth1.20 con IP 192.168.147.1 (gateway SubRed 2). Con IP forwarding habilitado, los nodos de diferentes VLANs pueden comunicarse pasando por el Admin VM.

**Capa 3 — Nodos de red:** SubRed 1 (VLAN 10, 192.168.56.0/24) contiene Arch Linux y Rocky Linux como VMs en VirtualBox usando el adaptador Host-Only #1, y Fedora como contenedor Docker Desktop en una red bridge con el mismo rango de subred. SubRed 2 (VLAN 20, 192.168.147.0/24) contiene Ubuntu como VM y Alpine y Kali Linux como contenedores Docker Desktop.

**Capa 4 — Captura de tráfico SPAN:** El switch espeja todo el tráfico de las VLANs 10 y 20 hacia el puerto e0/3, conectado al Cloud-SPAN (Ethernet 8). La VM Admin recibe ese tráfico en eth2 en modo promiscuo sin IP asignada. softflowd, pmacctd (sFlow) y nfprobe (IPFIX) leen de eth2 y generan registros de flujo hacia el colector pmacct local.

**Capa 5 — Recolección y almacenamiento:** pmacct actúa como colector central escuchando simultáneamente NetFlow v9 en puerto 2055, sFlow v5 en 6343 e IPFIX en 4739. Los flujos se almacenan en MySQL (base pmacct_db, tabla flows) con campos de src_host, dst_host, protocolo, puertos, bytes y paquetes.

**Capa 6 — Monitoreo y visualización:** Zabbix Server recolecta métricas de sistema de los seis nodos mediante agentes instalados en cada uno. Grafana consolida ambas fuentes (MySQL y Zabbix) en un dashboard unificado con paneles de tráfico, conversaciones y métricas de sistema.

---

### Pregunta 2: ¿Cuál es la relevancia de NetFlow, sFlow e IPFIX?

Los tres protocolos permiten realizar **teletráfico real** convirtiendo el tráfico de red en estadísticas analizables. Cada uno tiene enfoque diferente:

**NetFlow v9:** Creado por Cisco, exporta estadísticas de cada flujo TCP/UDP procesado. En este laboratorio softflowd lo genera desde el tráfico capturado por SPAN. Permite conocer exactamente qué pares de hosts se comunican, cuántos bytes intercambian y qué protocolos usan. Es fundamental para calcular la intensidad de tráfico (erlangs de datos) y detectar anomalías de comportamiento en la red.

**sFlow v5:** Trabaja por muestreo estadístico. En lugar de procesar cada paquete, toma una muestra de cada N paquetes y la exporta al colector. Es más escalable para redes de alta velocidad. pmacctd como probe sFlow lee directamente de eth2. Es relevante porque representa la técnica usada en switches de alta gama donde procesar cada flujo individual sería prohibitivo computacionalmente.

**IPFIX (RFC 7011):** Es la estandarización de NetFlow v9 por la IETF, haciendo el protocolo independiente del fabricante. Garantiza interoperabilidad entre equipos de diferentes marcas. En el proyecto, nfprobe de pmacct genera IPFIX desde la captura SPAN. Es importante en entornos reales donde coexisten switches Cisco, Juniper, Huawei, etc., todos exportando al mismo colector con el mismo protocolo estándar.

**Importancia conjunta:** Implementar los tres sobre la misma red permite comparar su granularidad, overhead de red generado y precisión de medición, dando una visión completa de las herramientas reales usadas en gestión de redes de producción.

---

### Pregunta 3: ¿Qué se obtiene con Wireshark? ¿Qué pasa cuando se ejecuta iPerf?

**Con Wireshark capturando en eth2 (SPAN):** Al capturar en la interfaz SPAN se obtiene una copia exacta y pasiva de todo el tráfico de ambas VLANs sin interrumpir el flujo normal de datos. Se obtiene visibilidad completa de cabeceras L2 a L7, streams TCP/UDP completos, resolución ARP entre subredes, y los propios registros de flujo de NetFlow/sFlow/IPFIX circulando por el loopback. También permite medir latencia real entre hosts (RTT) y detectar retransmisiones TCP.

**Cuando se ejecuta iPerf3:** En modo TCP, iPerf establece un handshake de tres vías visible en Wireshark (SYN → SYN-ACK → ACK) y luego satura el canal con segmentos TCP. Wireshark muestra el crecimiento de la ventana de congestión, los ACKs del receptor y eventuales retransmisiones. En modo UDP, iPerf envía datagramas a la tasa configurada sin control de flujo. Wireshark muestra todos los datagramas con sus números de secuencia propietarios de iPerf, y si hay congestión se observan gaps en la numeración. Simultáneamente, pmacct registra esos flujos en MySQL y Grafana actualiza los paneles de tráfico por segundo en tiempo real.

---

### Pregunta 4: Diferencias entre VMs y contenedores al instalar los SO

| Aspecto | VM VirtualBox | Contenedor Docker |
|---|---|---|
| Tiempo de instalación | 20-45 minutos (instalador completo del SO) | 30 segundos - 2 minutos (docker pull) |
| Tamaño en disco | 5-20 GB | 100 MB - 2 GB |
| Arranque | 30-90 segundos (boot completo) | Menos de 2 segundos |
| Kernel | Kernel propio aislado | Comparte kernel del host (WSL2 en Windows) |
| Proceso de instalación | Particiones, bootloader GRUB, /etc/fstab, systemd completo | No hay instalación: imagen pre-construida lista para usar |
| Persistencia | Disco virtual persistente por defecto | Efímero por defecto; requiere volúmenes para persistir datos |
| Configuración de red | Interfaz virtual independiente (eth0, eth1) participando en VLAN | Virtual bridge; integración con VLAN requiere configuración adicional |
| Consumo de RAM | 512 MB - 4 GB reservados | 10-200 MB adicionales por contenedor |
| Observación práctica | Al instalar Arch Linux se ve el proceso completo: particionado con fdisk/cfdisk, instalación de base con pacstrap, generación de fstab, instalación de GRUB, configuración de red desde cero | Al hacer `docker run fedora:latest` el contenedor está disponible inmediatamente con el SO funcionando; no hay proceso de instalación, no hay GRUB, PID 1 es el proceso indicado en el CMD |

**Conclusión práctica del laboratorio:** Las VMs simulan con mayor fidelidad equipos físicos reales (tienen su propio kernel, interfaz de red completa, participan directamente en las VLANs a través del adaptador Host-Only). Los contenedores son más eficientes y rápidos de desplegar pero requieren configuración adicional para integrarse en una topología de VLANs, ya que su networking se basa en bridges virtuales del host en lugar de adaptadores de red independientes.

---

## Referencias

- Cisco IOU L2 Configuration Guide
- pmacct Documentation — pmacct.net
- Zabbix 6.4 Documentation — zabbix.com/documentation/6.4
- Grafana Documentation — grafana.com/docs
- RFC 3954 — NetFlow v9
- RFC 3176 — sFlow v5
- RFC 7011 — IPFIX
- GNS3 Documentation — docs.gns3.com
- Docker Desktop Documentation — docs.docker.com/desktop/windows
