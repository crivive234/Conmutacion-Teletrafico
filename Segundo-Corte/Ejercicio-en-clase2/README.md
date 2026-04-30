# NetConverge Assistant

> Sistema modular de gestión y análisis en tiempo real para redes convergentes (voz, video, datos), con plantillas de configuración multi-vendor para Cisco IOS y Huawei VRP.

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 1. Resumen

**NetConverge Assistant** es una aplicación que asiste al ingeniero de redes en dos tareas que normalmente se hacen con herramientas separadas:

1. **Diseñar y generar configuración** para switches/routers Cisco y Huawei a partir de una intención de alto nivel (VLANs de voz, QoS, ACLs), con validación previa, *diff* contra la configuración anterior y *audit log*.
2. **Monitorear la calidad de la red en tiempo real**, capturando flujos RTP/RTSP y reportando MOS, jitter, latencia y pérdida de paquetes según las normas ITU-T G.107 y RFC 3550.

No requiere equipos físicos: la "aplicación" de configuraciones se simula escribiendo el script generado al log, y el tráfico de voz/video puede inyectarse sintéticamente. Esto lo hace ideal para entornos académicos, pruebas de concepto y diseño de arquitecturas antes de tocar hardware real.

---

## 2. Arquitectura

El proyecto sigue *Clean Architecture* en tres capas, con un sistema de plugins para soportar nuevos vendors sin modificar el núcleo:

```
┌──────────────────────────────────────────────────────────────────┐
│  interfaces/         CLI (Typer + Rich)  ·  Web (FastAPI + React)│
├──────────────────────────────────────────────────────────────────┤
│  application/        Casos de uso                                 │
│                      · GenerateConfig      · AnalyzeRTPStream     │
│                      · ApplyConfigSim      · RaiseAlert           │
│                      · ComplianceCheck     · GenerateReport       │
├──────────────────────────────────────────────────────────────────┤
│  domain/             Modelos puros (Pydantic) y reglas            │
│                      · VoiceAccessPort, QoSPolicy, ACL            │
│                      · StreamMetrics, MOSReport, AuditEvent       │
├──────────────────────────────────────────────────────────────────┤
│  infrastructure/     Adaptadores concretos                        │
│  ├─ vendors/         · CiscoIOS  · HuaweiVRP  (plantillas Jinja2) │
│  ├─ capture/         · ScapyRTPCapture  · SyntheticGenerator      │
│  ├─ persistence/     · SQLite + SQLAlchemy (configs, eventos)     │
│  └─ transport/       · WebSocket telemetry  · REST API            │
└──────────────────────────────────────────────────────────────────┘
```

**Principios de diseño**

- El **dominio no conoce** Jinja2, Scapy, ni FastAPI. Los modelos son `BaseModel` puros y los casos de uso reciben adaptadores por inyección.
- Los **adaptadores de vendor** implementan una interfaz `VendorAdapter` común. Añadir un vendor nuevo = una subclase + un directorio de plantillas. La interfaz es deliberadamente similar a la de NAPALM para permitir migración futura a equipos reales.
- El **renderizado de configuración es offline y determinista**. La "aplicación" al equipo es una operación separada que, en este proyecto, se simula. Esto permite revisión, *peer review* y *dry-run* antes de cualquier *push*.

---

## 3. Módulos

### 3.1 Generador de configuración multi-vendor

A partir de un modelo de intención (Pydantic), genera configuración válida para cada vendor. Cubre los casos del ejercicio:

| Función | Cisco IOS | Huawei VRP |
|---|---|---|
| Puerto de acceso con VLAN voz | `switchport voice vlan` + `mls qos trust cos` | `voice-vlan <id> enable` + `trust 8021p` |
| QoS con LLQ/PQ-WFQ | `class-map` → `policy-map` (`priority`, `bandwidth`) | `traffic classifier` → `traffic behavior` (`queue ef`, `queue af`, `queue wfq`) |
| ACL extendida | `ip access-list extended <name>` con permit/deny por puerto | `acl name <name> advance` con `rule <n> permit/deny` |

**Marcado DSCP por defecto** según RFC 4594:

- **EF (46)** — voz RTP
- **AF41 (34)** — video conferencia
- **AF31 (26)** — video streaming
- **CS3 (24)** — señalización (SIP, H.323)
- **CS6 (48)** — control de red

### 3.2 Telemetría y dashboards

- Métricas sintéticas con `asyncio` o capturadas con Scapy.
- Almacenamiento en SQLite particionado por hora (o InfluxDB opcional).
- Tres vistas en el dashboard:
  - **Disponibilidad**: uptime %, latencia ICMP, estado de interfaces.
  - **Rendimiento**: throughput por interfaz, jitter, *packet loss*, ancho de banda utilizado.
  - **Eventos**: syslog, umbrales superados, alertas activas.
- WebSocket bidireccional al frontend para refresco a 1 Hz sin polling.

**Umbrales por defecto** (ITU-T G.114 + RFC 4594):

| Métrica | Voz | Video |
|---|---|---|
| One-way delay | < 150 ms (aceptable) / < 400 ms (máximo) | < 150 ms |
| Jitter | < 30 ms | < 50 ms |
| Packet loss | < 1 % | < 0.1 % |

### 3.3 Analizador RTP en tiempo real

- Captura paquetes RTP (UDP) y RTSP/RTP (control TCP, datos UDP) usando Scapy con `scapy.contrib.rtp`.
- Soporta payload types 0 y 8 (G.711) y 18 (G.729A).
- Calcula:
  - **Jitter** según RFC 3550 §6.4.1 con la fórmula recursiva `J = J + (|D(i-1,i)| - J)/16`.
  - **Pérdida** desde *sequence numbers* RTP, manejando *wrap-around* a 16 bits y *out-of-order*.
  - **MOS** con el modelo E (G.107) simplificado:
    - R₀ = 93.2 (G.711)
    - I_d a partir del *one-way delay* efectivo (delay base + 2× jitter para absorción del *jitter buffer*)
    - I_e-eff con el modelo de PLC (B_pl = 25.1 para G.711)
    - MOS = 1 + 0.035·R + 7·10⁻⁶·R·(R−60)·(100−R)
- Genera alertas cuando MOS < 3.6 o jitter > 30 ms sostenido por 5 s.

### 3.4 Persistencia y auditoría

- **Configuraciones**: cada render se guarda con timestamp, autor, vendor, intención, *output* y estado (`pending`/`applied_simulated`/`rolled_back`).
- **Audit log inmutable**: toda acción (login, render, *push* simulado, modificación de umbrales) queda registrada con SHA-256 encadenado para detección de manipulación.
- **Snapshots**: el histórico de configuraciones permite generar *diffs* entre cualquier par de versiones.

### 3.5 Compliance check (módulo opcional)

Recibe una configuración Cisco existente (texto crudo), la parsea con `ciscoconfparse2` y reporta violaciones de política:

- Interfaces de acceso sin `spanning-tree portfast` o sin `bpduguard`.
- ACLs con `permit ip any any`.
- Comunidades SNMP por defecto (`public`/`private`).
- Líneas VTY sin ACL aplicada.

Para Huawei se implementa un parser simplificado basado en el formato de bloques `#`.

---

## 4. Stack tecnológico

| Capa | Herramientas |
|---|---|
| Lenguaje | Python 3.11+ |
| Backend API | FastAPI + Uvicorn |
| Frontend | React + Vite + Recharts (alternativa: Plotly Dash si se quiere monorepo Python) |
| CLI | Typer + Rich |
| Plantillas | Jinja2 |
| Validación | Pydantic v2 |
| Captura de red | Scapy (con `scapy.contrib.rtp`) |
| Parsing de configs | ciscoconfparse2 |
| Persistencia | SQLAlchemy 2 + SQLite (InfluxDB opcional) |
| Telemetría real-time | WebSockets (FastAPI nativo) |
| Tests | pytest + pytest-asyncio |
| Empaquetado | Docker + docker-compose |

**No se usa** GNS3, EVE-NG ni Cisco CML. La simulación es propia y consiste en:

- Escribir el script de configuración generado a un log marcado como `applied_simulated`.
- Generar tráfico RTP sintético con parámetros controlables (pérdida, jitter, codec) usando Scapy en modo *raw socket*.

---

## 5. Estructura del proyecto

```
netconverge/
├── docker-compose.yml
├── pyproject.toml
├── README.md
├── domain/
│   ├── models/
│   │   ├── intents.py          # VoiceAccessPort, QoSPolicy, ACL, DSCP
│   │   ├── telemetry.py        # StreamMetrics, MOSReport
│   │   └── audit.py            # AuditEvent
│   └── thresholds.py           # constantes ITU-T / RFC
├── application/
│   ├── config/
│   │   ├── generate.py         # caso de uso GenerateConfig
│   │   └── apply_simulated.py
│   └── analysis/
│       ├── rtp_quality.py      # RTPStreamAnalyzer (E-model + jitter)
│       └── alert_engine.py
├── infrastructure/
│   ├── vendors/
│   │   ├── base.py             # VendorAdapter (ABC)
│   │   ├── cisco_ios.py
│   │   ├── huawei_vrp.py
│   │   └── templates/
│   │       ├── cisco_ios/
│   │       │   ├── voice_port.j2
│   │       │   ├── qos_policy.j2
│   │       │   └── acl.j2
│   │       └── huawei_vrp/
│   │           ├── voice_port.j2
│   │           ├── qos_policy.j2
│   │           └── acl.j2
│   ├── capture/
│   │   ├── scapy_rtp.py
│   │   └── synthetic.py        # generador de tráfico sintético
│   ├── persistence/
│   │   ├── models.py
│   │   └── repositories.py
│   └── transport/
│       ├── api.py              # FastAPI app
│       └── websocket.py
├── interfaces/
│   ├── web/                    # frontend React
│   └── cli/
│       └── main.py             # netconverge ... (Typer)
├── tests/
│   ├── unit/
│   │   ├── test_intents.py
│   │   ├── test_cisco_render.py
│   │   ├── test_huawei_render.py
│   │   └── test_mos_calculation.py
│   ├── integration/
│   └── fixtures/
│       └── rtp_captures/       # .pcap de referencia
└── docs/
    ├── architecture.md
    └── vendor_command_reference.md
```

---

## 6. Instalación

### Requisitos

- Python 3.11+
- Docker y docker-compose (recomendado)
- Para captura real: privilegios de *raw socket* (`sudo` en Linux, *Npcap* en Windows)

### Vía Docker (recomendado)

```bash
git clone https://github.com/<your-user>/netconverge.git
cd netconverge
docker-compose up --build
```

Esto levanta tres servicios:

- `backend` — FastAPI en `:8000`
- `frontend` — React en `:5173`
- `traffic-gen` — generador de RTP sintético que apunta al backend

### Vía pip (desarrollo)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
netconverge --help
```

---

## 7. Uso

### 7.1 CLI

**Generar configuración para un puerto de acceso con VLAN voz (Cisco):**

```bash
netconverge config voice-port \
  --vendor cisco_ios \
  --interface GigabitEthernet0/1 \
  --data-vlan 10 --voice-vlan 20 \
  --description "Phone + PC daisy chain"
```

Salida:

```
interface GigabitEthernet0/1
 description Phone + PC daisy chain
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!

[saved] config_id=42 status=pending
```

**Aplicar configuración (simulado) y registrar en audit log:**

```bash
netconverge config apply 42 --device-name SW-CAMPUS-01
```

**Lanzar el analizador RTP sobre tráfico sintético con 3% de pérdida:**

```bash
netconverge analyze synth --packets 1000 --loss 0.03 --jitter-ms 5
```

Salida:

```
SSRC 0xC0FFEE | codec G.711  packets=970/1000 loss=3.0%
                jitter=5.2ms   delay≈110ms
                R=76.4  MOS=3.78  status=BUENO
```

**Compliance check sobre un archivo de configuración existente:**

```bash
netconverge audit running-config.txt --policy default
```

### 7.2 Web UI

Abrir `http://localhost:5173`. La UI tiene cuatro pestañas: **Configurar**, **Dashboard**, **Análisis RTP**, **Auditoría**.

### 7.3 API REST (extracto)

```
POST /api/v1/config/render            { vendor, intent_type, intent_data } → config text + diff
POST /api/v1/config/{id}/apply         → simula push y registra audit event
GET  /api/v1/streams                   → lista de SSRCs activos
GET  /api/v1/streams/{ssrc}/metrics    → snapshot de MOS, jitter, loss
WS   /ws/telemetry                     → stream de métricas a 1 Hz
```

---

## 8. Escenarios de demostración

El proyecto incluye tres demos ejecutables con `make demo-<name>` o `netconverge demo <name>`:

### Demo 1 — Red sana

10 minutos de tráfico RTP G.711 con jitter de fondo de 2 ms y 0% de pérdida. **Resultado esperado:** MOS sostenido > 4.2, sin alertas.

### Demo 2 — Degradación de jitter

Inyecta jitter creciente de 5 → 80 ms a lo largo de 5 minutos. **Resultado esperado:** alerta `JITTER_HIGH` a partir del minuto 2, MOS cae de 4.3 a ~3.0, dashboard muestra el patrón en la gráfica de jitter.

### Demo 3 — ACL bloqueando SIP

Genera la ACL `VOICE-PERMIT` para Cisco, la "aplica" simulada, luego intenta enviar tráfico SIP en puerto 5061 (no incluido en la ACL). **Resultado esperado:** el simulador rechaza el flujo, el log muestra denial, el dashboard registra el evento. Demuestra el ciclo intención → configuración → aplicación → verificación.

---

## 9. Referencias normativas

- **RFC 3550** — RTP: A Transport Protocol for Real-Time Applications (cálculo de jitter, §6.4.1)
- **RFC 4594** — Configuration Guidelines for DiffServ Service Classes (mapeo DSCP)
- **ITU-T G.107** (06/2015) — The E-model: a computational model for use in transmission planning (MOS)
- **ITU-T G.114** — One-way transmission time (umbrales de delay)
- **ITU-T G.711 / G.729** — codecs y sus parámetros de impairment
- **Cisco IOS QoS Configuration Guide** — sintaxis MQC (`class-map`, `policy-map`, `service-policy`)
- **Huawei VRP Configuration Guide — QoS** — sintaxis `traffic classifier`/`behavior`/`policy`
- **Huawei VRP Voice VLAN Configuration** — comandos `voice-vlan enable` y `voice-vlan mode`

---

## 10. Roadmap

- [ ] Soporte para Juniper Junos (`set` style + commit confirmed)
- [ ] Modo "live" con NAPALM para conexión a equipos reales
- [ ] Importación de capturas `.pcap` para análisis offline
- [ ] Modelo E completo (G.107 con eco, codecs adicionales, advantage factor)
- [ ] Plantillas para *port-security* y *DHCP snooping*
- [ ] Integración con Grafana vía datasource SQLite

---

## 11. Limitaciones honestas

- El **modelo E** implementado es la versión simplificada para G.711/G.729 sin eco. No es sustituto de un E-model completo certificado para planeación de redes a escala carrier.
- La **simulación de aplicación** no detecta errores que solo aparecerían en un equipo real (conflictos de VLAN, falta de licencia QoS, plataformas que no soportan ciertos comandos).
- El **compliance check** parsea sintaxis, no semántica completa: una ACL `permit ip 10.0.0.0 0.0.0.255 any` se considera válida aunque sea funcionalmente equivalente a `permit ip any any` en una topología específica.
- El **generador de tráfico sintético** produce paquetes RTP estructuralmente correctos pero el payload es ruido, no audio real.

---

## 12. Licencia

MIT. Ver `LICENSE`.

## 13. Créditos

Proyecto académico desarrollado como parte del curso de Redes Convergentes. Inspirado en el ecosistema de automatización de Python para redes (Netmiko, NAPALM, Nornir, ciscoconfparse2) y en la documentación pública de Cisco y Huawei.
