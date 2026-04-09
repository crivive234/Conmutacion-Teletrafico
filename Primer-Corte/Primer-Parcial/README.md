# Primer Parcial - Conmutación y Teletráfico

**Cristian David Viasus Vega**

---

## Primer Punto — Conceptual

### a) Latencia vs. Jitter en VoIP

La latencia es el tiempo total que tarda un paquete en viajar desde el origen hasta el destino y se mide en milisegundos (ms), además es un valor absoluto que puede ser alto pero constante. En cambio el jitter es la variación en el retardo de llegada entre paquetes consecutivos.

El jitter tiene mayor impacto negativo en VoIP. Aunque una latencia alta de aproximadamente 150 ms es tolerable, el jitter destruye la calidad de la llamada porque los paquetes de audio llegan de forma irregular, generando cortes o interferencia en el sonido.

### b) TCP vs. UDP para transmisión de video

UDP es más eficiente en throughput porque la cabecera UDP tiene solo 8 bytes. No tiene mecanismos de control, no hay confirmaciones (ACKs), no hay retransmisiones, y no hay control de flujo. Esto maximiza el throughput disponible para los datos de video.

En cambio si uso TCP tengo mayor control de pérdida de paquetes pero es más lento porque la cabecera TCP tiene entre 20 a 60 bytes. Los campos de la cabecera permiten detectar paquetes perdidos y retransmitirlos, garantizando entrega confiable y ordenada. Sin embargo, para el streaming en vivo, las retransmisiones llegan tarde.

### c) Protocolo ARP y la tabla `arp -a`

El protocolo ARP llena la tabla que muestra `arp -a`. Su función principal es resolver o mapear una dirección IP conocida a una dirección MAC (física) dentro de la red local (LAN).

La relación con la trama Ethernet es que requiere obligatoriamente una MAC de destino en su cabecera que por su anatomía tiene los campos: MAC Origen y Destino cada una de 6 bytes, Tipo (2 bytes), Datos y FCS.

Cuando el host conoce la IP destino pero no su MAC, envía un broadcast ARP y el host dueño de esa IP responde con su MAC. Así el equipo puede construir la trama Ethernet correctamente y enviarla.

### d) SNMPv2c vs. SNMPv3

SNMPv2c sólo usa texto plano, no hay autenticación y cifrado. Cualquiera que capture el tráfico puede leer el texto plano. En cambio SNMPv3 implementa autenticación (MD5/SHA) y cifrado (DES/AES). Los mensajes viajan cifrados y autenticados.

Los tipos de mensajes en SNMPv2c son GetRequest, SetRequest, Response, InformRequest… En cambio SNMPv3 usa los mismos tipos de mensajes pero encapsulados con cabecera de seguridad y parámetros de autenticación.

### e) OID, MIB y operaciones SNMP

Un OID es un identificador único que apunta a un objeto específico dentro de la MIB y se representa como una cadena de números separados por puntos. La MIB es la base de datos estructurada en árbol que define todos los objetos gestionables de un dispositivo y sus OIDs asociados.

En SNMP, el administrador debe usar una operación **GetRequest**, enviando el OID al agente SNMP del dispositivo. El agente responde con el valor actual del contador.

Un Trap es un mensaje que el agente envía al gestor cuando ocurre un evento predefinido (umbral, fallo, etc.). No sirve para consultas periódicas. El agente no responde a preguntas del gestor. Es unidireccional porque si el administrador quiere saber cuántos bytes ha recibido una interfaz en este momento, debe preguntar activamente con un Get.

---

## Segundo Punto — Análisis de Captura con Wireshark

### a) Cabecera Ethernet y campo Tipo 0x0800

- **MAC Destino:** Es la dirección física del receptor en la LAN.
- **MAC Origen:** Dirección física del emisor.
- **Tipo / EtherType (0x0800):** Indica el protocolo de capa superior encapsulado.

El valor `0x0800` significa que el payload de la trama Ethernet contiene un paquete IPv4. Hay otros valores comunes como `0x8100` en VLAN (802.1Q).

### b) Campos Protocolo y TTL en IPv4

El valor 6 (TCP) indica qué protocolo de la capa de transporte está encapsulado dentro del paquete IP. Este campo es fundamental para que el receptor sepa cómo procesar el payload.

El TTL (Time To Live) es un contador que se decrementa en 1 por cada salto que atraviesa el paquete. Cuando llega a 0, el router descarta el paquete y envía un mensaje ICMP de tiempo excedido al origen.

Su importancia radica en evitar que paquetes con rutas incorrectas circulen indefinidamente por la red (bucles de enrutamiento). El valor 128 indica que el paquete salió de Windows, cuyo TTL inicial estándar es 128, y aún no ha pasado por ningún router.

### c) Flags ACK y PSH, y Puerto Destino 80

- **Flag ACK (Acknowledgment):** Ayuda a que el receptor confirme que ha recibido correctamente todos los bytes hasta ese número de secuencia.
- **Flag PSH (Push):** Le indica a la pila TCP del receptor que entregue inmediatamente los datos a la aplicación, sin esperar a llenar el buffer. Esto es típico en aplicaciones interactivas donde la respuesta inmediata es importante.

El puerto 80 es el puerto asignado al protocolo HTTP. El campo Datos de segmento lo confirma e indica que se está intentando acceder a un servidor web (una instrucción típica de un navegador solicitando la página principal de un sitio). Si fuera 443, sería HTTPS sobre TLS.

### d) IPv6 vs. IPv4 en cabecera

En IPv6, la cabecera base de 40 bytes reemplaza a la cabecera IPv4 que es variable entre 20 a 60 bytes. La mejora más notable para los routers es la eliminación del campo de checksum en la cabecera IP.

En IPv4, cada router debe recalcular el checksum tras decrementar el TTL. En IPv6 esto desaparece completamente, lo que reduce la carga de procesamiento en cada salto. Además, IPv6 elimina la fragmentación en routers y solo el origen fragmenta, lo que simplifica y acelera el reenvío de paquetes.

---

## Tercer Punto — Diagnóstico y Gestión con Herramientas de Windows

### a) Comando `pathping`

El `pathping` combina las funcionalidades de `ping` y `tracert` en un solo comando, pero añade estadísticas de pérdida de paquetes por cada salto (hop). Mientras `tracert` solo muestra la latencia en cada salto y `ping` solo mide conectividad al destino final, `pathping` permite identificar exactamente en qué router intermedio se están perdiendo paquetes y en qué porcentaje.

### b) Monitoreo SNMP del router

En Windows se puede usar la herramienta `snmpwalk.exe` o `snmputil.exe`. El comando para "caminar" por el árbol MIB sería el siguiente:

```
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.2.1.2
```

Esto "camina" por el subárbol de interfaces MIB obteniendo todos los valores de todas las interfaces del router como velocidad, estado operacional, bytes transmitidos/recibidos, errores, etc.

Un mensaje Trap `authenticationFailure` se genera cuando el agente SNMP del router recibe una consulta con una community string incorrecta, es decir, que alguien intentó acceder con la contraseña equivocada. Es básicamente un aviso de acceso no autorizado.

Con polling, el gestor debe consultar el router periódicamente aunque no haya ningún problema, consumiendo recursos. En cambio con Trap, el router solo envía un mensaje cuando ocurre el evento, lo que libera al gestor de consultas innecesarias. El gestor reacciona solo cuando hay algo que atender, lo que permite una respuesta más rápida al problema real.

---

## Cuarto Punto — El Viaje de Commit a GitHub

### Paso 1 — Verificación de conectividad básica y resolución de nombres

**Pregunta 1**

Para verificar la conectividad IP hacia GitHub utilizaría el comando:

```
ping github.com
```

`ping` usa el protocolo ICMP (Internet Control Message Protocol) de la capa de red y envía mensajes Echo Request esperando Echo Reply. Verifica que existe conectividad IP entre el equipo local y el servidor de GitHub, midiendo el RTT (Round-Trip Time).

**Pregunta 2**

Cuando se escribe `github.com`, el SO consulta al servidor DNS configurado y el proceso que sigue es el siguiente:

1. Hace una consulta local en la caché DNS.
2. Si no está, envía un query UDP al puerto 53 del servidor DNS.
3. El servidor DNS responde con la IP de `github.com`.

Si la resolución fallara, se diagnostica con el comando:

```
nslookup github.com
```

**Pregunta 3**

`git push` usa TCP que es sensible a la latencia alta: si eso pasa, los ACKs tardan más en regresar y TCP envía datos más lentamente. Con jitter, puede interpretar la variabilidad (retardo de paquetes) como pérdida de paquetes, activando su mecanismo de control y reduciendo el tamaño de la ventana de transmisión.

En términos prácticos el `git push` no fallará, pero se ejecutará más lento de lo esperado. Si falla, es porque el timeout de la conexión expira.

---

### Paso 2 — Establecimiento de la conexión para el push

**Pregunta 1**

Git usa el protocolo TCP para garantizar que los datos del push lleguen completos y en orden. El three-way handshake establece la conexión de la siguiente manera:

1. **SYN:** El cliente envía un segmento con el flag SYN activado a GitHub, indicando que quiere iniciar una conexión.
2. **SYN-ACK:** GitHub responde confirmando con SYN+ACK, aceptando la conexión.
3. **ACK:** El equipo confirma con un ACK final.

**Pregunta 2**

Para observar los segmentos TCP en tiempo real se usa Wireshark, aplicando el siguiente filtro (asumiendo que la IP de GitHub es `140.82.114.3`):

```
ip.addr == 140.82.114.3 && tcp
```

Esto muestra exclusivamente el tráfico TCP hacia y desde los servidores de GitHub, incluyendo el handshake, los datos del push y el cierre de conexión.

**Pregunta 3**

Los puertos identificados son:
- **Puerto Destino: 443**, ya que `git push` usa HTTPS cifrado con TLS sobre TCP.
- **Puerto Origen:** aleatorio, asignado por el SO al cliente.

La **capa de transporte (Capa 4)** gestiona estos puertos para multiplexar conexiones de diferentes aplicaciones.

---

### Paso 3 — Encapsulamiento y enrutamiento de los datos

**Pregunta 1**

Desde que Git genera los datos hasta que salen por la tarjeta de red, cada capa agrega su propia cabecera. El proceso de encapsulamiento (PDUs por capa) es el siguiente:

| Capa OSI       | PDU            | Qué ocurre                                                        |
|----------------|----------------|-------------------------------------------------------------------|
| Aplicación (7) | Mensaje/Datos  | Git empaqueta el commit en formato HTTP/HTTPS                     |
| Transporte (4) | Segmento       | TCP agrega cabecera con puertos, número de secuencia, flags       |
| Red (3)        | Paquete        | IP agrega direcciones origen (tu PC) y destino (GitHub)           |
| Enlace (2)     | Trama          | Ethernet agrega MACs origen y destino + tipo 0x0800               |
| Física (1)     | Bits           | La trama se convierte en señal eléctrica/óptica y sale por la NIC |

**Pregunta 2**

Si un router intermedio descarta paquetes, el `git push` se volvería extremadamente lento o podría interrumpirse temporalmente. El mecanismo de TCP que se activaría es el **Control de Congestión**, que funciona así:

1. TCP detecta la pérdida porque no llega el ACK esperado.
2. Reduce drásticamente el tamaño de la ventana de transmisión.
3. Comienza a reenviar los segmentos perdidos.
4. Reinicia el crecimiento de la ventana lentamente.

Esto reduce el throughput efectivo hasta recuperarse. El comando para identificar el salto con pérdida es:

```
pathping github.com
```

Muestra el % de pérdida por salto, permitiendo localizar exactamente el router problemático.

**Pregunta 3**

El campo que evita los bucles infinitos en la cabecera IP es el **TTL (Time To Live)**. Su funcionamiento es simple pero crítico:

1. El emisor establece un valor inicial (típicamente 128).
2. Cada router que reenvía el paquete decrementa el TTL en 1.
3. Si el TTL llega a 0, el router descarta el paquete y envía un mensaje ICMP "Time Exceeded" de vuelta al origen.
4. Esto evita que paquetes perdidos o mal enrutados circulen indefinidamente consumiendo ancho de banda.

Esto mismo aprovecha `tracert` para descubrir los saltos: envía paquetes con TTL=1, 2, 3... y va recolectando los mensajes ICMP de cada router.

---

### Paso 4 — Confirmación y fin de la comunicación

**Pregunta 1**

El mensaje que usa es **ACK (Acknowledgment)** para confirmar la recepción correcta de los datos. Su relación con la fiabilidad y pérdida de paquetes es directa:

1. TCP asigna un número de secuencia a cada segmento enviado.
2. GitHub responde con un ACK indicando el siguiente byte que espera recibir.
3. Si el equipo no recibe un ACK dentro del tiempo límite (Retransmission Timeout), asume que el paquete se perdió y lo retransmite automáticamente.

Esto es precisamente lo que hace a TCP confiable: ningún dato se da por entregado sin su ACK correspondiente, garantizando que el commit llegue completo e íntegro a GitHub.

**Pregunta 2**

Una vez completado el push, la conexión se cierra de la siguiente manera:

| Paso | Actor      | Flag | Significado                 |
|------|------------|------|-----------------------------|
| 1    | mi equipo  | FIN  | "Terminé de enviar datos"   |
| 2    | GitHub     | ACK  | "Entendido, recibí tu FIN"  |
| 3    | GitHub     | FIN  | "Yo también terminé"        |
| 4    | mi equipo  | ACK  | "Confirmado, cerramos"      |

A diferencia del Three-Way Handshake de apertura, el cierre requiere 4 pasos porque cada extremo cierra su lado de la conexión de forma independiente.

**Pregunta 3**

Como administrador, si quiero realizar monitoreo con SNMP durante el push, las métricas observables en el agente SNMP del router de salida serían las siguientes:

| OID / Métrica MIB | Qué indica                                              |
|-------------------|---------------------------------------------------------|
| ifOutOctets       | Bytes transmitidos por la interfaz (el volumen del push)|
| ifInOctets        | Bytes recibidos (ACKs de GitHub)                        |
| ifOutDiscards     | Paquetes descartados por congestión en el router        |
| ifOutErrors       | Errores en la transmisión                               |
| ifInUcastPkts     | Cantidad de paquetes unicast entrantes                  |

Para consultar estas métricas en tiempo real desde Windows con Net-SNMP se usa el comando:

```
snmpwalk -v 2c -c public 192.168.1.1 ifOutOctets
```

La versión que usaría es **SNMPv3**, ya que es la única versión que ofrece:
- **Autenticación:** verifica que el gestor es quien dice ser.
- **Cifrado:** protege el contenido de las consultas.
- **Control de acceso:** define qué usuario puede leer/escribir qué OIDs.

SNMPv2c, como se usó en el punto anterior, transmite la community string `public` en texto plano, lo que representa un riesgo de seguridad inaceptable para un entorno de producción.