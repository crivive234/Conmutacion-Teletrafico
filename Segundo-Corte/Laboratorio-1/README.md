# 9.3.3 Packet Tracer – Guía de Configuración HSRP

## 📋 Descripción

Se trabaja la configuración del protocolo **HSRP (Hot Standby Router Protocol)**, que pertenece a la familia **FHRP** (*First Hop Redundancy Protocols*) y permite ofrecer **puertas de enlace predeterminadas redundantes** a los hosts de una LAN, garantizando conectividad continua ante la falla de un router gateway.

---

## 🎯 Objetivos

- Configurar un **router activo** HSRP.
- Configurar un **router en espera** HSRP.
- Verificar y observar la operación de HSRP ante fallos de enlace.

---

## 🗺️ Topología de Red

![Topología](/Segundo-Corte/Laboratorio-1/capturas/Topologia.png)

---

## 📊 Tabla de Asignación de Direcciones

![Direcciones](/Segundo-Corte/Laboratorio-1/capturas/Direcciones.png)


> **Nota:** El router I-Net está presente en la nube de Internet y no se puede acceder en esta actividad.

---

## 🔧 Desarrollo de la Actividad

---

### PARTE 1: Verificar la Conectividad

#### Paso 1 – Trazar la ruta desde PC-A al servidor web

Desde el símbolo del sistema de **PC-A** se ejecuta:

```
tracert 209.165.200.226
```

> ✅ **Respuesta:**  
> Los dispositivos en la ruta desde PC-A al servidor web son:  
> **R1 → R2 → I-Net**  
> *(PC-A usa R1 como gateway, luego el tráfico pasa por R2 y finalmente por I-Net hacia el servidor)*

---

#### Paso 2 – Trazar la ruta desde PC-B al servidor web

Desde el símbolo del sistema de **PC-B** se ejecuta:

```
tracert 209.165.200.226
```

> ✅ **Respuesta:**  
> Los dispositivos en la ruta desde PC-B al servidor web son:  
> **R3 → R2 → I-Net**  
> *(PC-B usa R3 como gateway, luego el tráfico pasa por R2 y finalmente por I-Net hacia el servidor)*

---

#### Paso 3 – Observar el comportamiento cuando R3 no está disponible

**Procedimiento:**
1. Se elimina el vínculo entre **R3** y **S3** usando la herramienta de eliminación de Packet Tracer.
2. Desde **PC-B** se ejecuta: `tracert 209.165.200.226`
3. Se compara el resultado con el del Paso 2.

> ✅ **Respuesta:**  
> El comando `tracert` **no puede determinar ninguna ruta** hacia el servidor web.  
> Todos los saltos muestran tiempo de espera agotado (`* * *`), lo que indica que PC-B ha perdido completamente la conectividad hacia el exterior.  
> A diferencia del Paso 2 (donde la ruta pasaba correctamente por R3), ahora **no existe ninguna ruta alternativa** para PC-B, ya que su único gateway configurado (R3) dejó de estar accesible.

4. Se restablece el enlace: clic en **Connections** → seleccionar **Copper Straight-Through** → conectar **S3 GigabitEthernet0/2** con **R3 GigabitEthernet0/0**.
5. Una vez que las luces del enlace estén en verde, se verifica con ping al servidor web → **el ping es exitoso**.

---

### PARTE 2: Configurar los Routers Activo y en Espera HSRP

#### Paso 1 – Configurar HSRP en R1 (Router Activo)

```bash
R1> enable
R1# configure terminal
R1(config)# interface g0/1
R1(config-if)# standby version 2
R1(config-if)# standby 1 ip 192.168.1.254
R1(config-if)# standby 1 priority 150
R1(config-if)# standby 1 preempt
R1(config-if)# end
```

**Explicación de cada comando:**

| Comando | Descripción |
|---|---|
| `interface g0/1` | Ingresa a la interfaz LAN de R1 (conectada a S1 / LAN 1) |
| `standby version 2` | Activa la versión 2 de HSRP (compatible con IPv4 e IPv6) |
| `standby 1 ip 192.168.1.254` | Define la **IP virtual** del grupo HSRP 1, que usarán los hosts como gateway |
| `standby 1 priority 150` | Asigna prioridad **150** (mayor al valor por defecto de 100), designando a R1 como router activo |
| `standby 1 preempt` | Permite que R1 **recupere automáticamente** el rol activo cuando vuelva a estar disponible |

> ✅ **Respuesta a la pregunta:**  
> ¿Cuál será la prioridad HSRP de R3 cuando se agregue al grupo HSRP 1?  
> **La prioridad de R3 será 100**, que es el valor por defecto. Como no se configura una prioridad explícita en R3, este valor es inferior al de R1 (150), por lo que R3 quedará automáticamente como router en espera.

---


#### Paso 2 – Configurar HSRP en R3 (Router en Espera)

Se configura únicamente la versión y la IP virtual (sin prioridad ni preempt):

```bash
R3> enable
R3# configure terminal
R3(config)# interface g0/0
R3(config-if)# standby version 2
R3(config-if)# standby 1 ip 192.168.1.254
R3(config-if)# end
```

> **Nota:** La interfaz usada en R3 es **G0/0** (conectada a LAN 2 / S3), a diferencia de R1 que usa G0/1.

---

#### Paso 3 – Verificar la configuración HSRP

**En R1:**

```bash
R1# show standby
```

```
GigabitEthernet0/1 - Group 1 (version 2)
  State is Active
    4 state changes, last state change 00:00:30
  Virtual IP address is 192.168.1.254
  Active virtual MAC address is 0000.0C9F.F001
    Local virtual MAC address is 0000.0C9F.F001 (v2 default)
  Hello time 3 sec, hold time 10 sec
    Next hello sent in 1.696 secs
  Preemption enabled
  Active router is local
  Standby router is 192.168.1.3
  Priority 150 (configured 150)
  Group name is "hsrp-Gi0/1-1" (default)
```
![R1 HSRP](/Segundo-Corte/Laboratorio-1/capturas/R1.jpg)

**En R3:**

```bash
R3# show standby
```

```
GigabitEthernet0/0 - Group 1 (version 2)
  State is Standby
    4 state changes, last state change 00:02:29
  Virtual IP address is 192.168.1.254
  Active virtual MAC address is 0000.0C9F.F001
    Local virtual MAC address is 0000.0C9F.F001 (v2 default)
  Hello time 3 sec, hold time 10 sec
    Next hello sent in 0.720 secs
  Preemption disabled
  Active router is 192.168.1.1
    MAC address is d48c.b5ce.a0c1
  Standby router is local
  Priority 100 (default 100)
  Group name is "hsrp-Gi0/0-1" (default)
```

![R3 HSRP](/Segundo-Corte/Laboratorio-1/capturas/R3.png)

> ✅ **Respuestas basadas en la salida del comando:**
>
> **¿Cuál router es el router activo?**  
> **R1** es el router activo (`State is Active` / `Active router is local`).
>
> **¿Cuál es la dirección MAC de la dirección IP virtual?**  
> **0000.0C9F.F001** — es la MAC virtual compartida por ambos routers en el grupo HSRP.
>
> **¿Cuál es la dirección IP y la prioridad del router en espera?**  
> Dirección IP: **192.168.1.3** (R3) — Prioridad: **100** (valor por defecto).

---

**Resumen rápido con `show standby brief`:**

```bash
R1# show standby brief
```
```
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/1         1  150 P Active   local           192.168.1.3     192.168.1.254
```


```bash
R3# show standby brief
```
```
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0         1  100   Standby  192.168.1.1     local           192.168.1.254
```

---

#### Cambiar el gateway predeterminado en hosts y switches

> ✅ **Respuesta:**  
> La dirección que deben usar todos los dispositivos como gateway es **192.168.1.254** (la IP virtual HSRP).

Configuración en los switches:

```bash
# Switch S1
S1> enable
S1# configure terminal
S1(config)# ip default-gateway 192.168.1.254
S1(config)# end

# Switch S3
S3> enable
S3# configure terminal
S3(config)# ip default-gateway 192.168.1.254
S3(config)# end
```

En **PC-A** y **PC-B**: cambiar manualmente la puerta de enlace predeterminada de `192.168.1.1` / `192.168.1.3` a `192.168.1.254`.

> ✅ **¿Los pings desde PC-A y PC-B al servidor web son exitosos?**  
> **Sí**, los pings son exitosos una vez que los hosts apuntan a la IP virtual HSRP como gateway.

---

### PARTE 3: Observar la Operación HSRP

#### Paso 1 – Verificar la nueva ruta desde PC-B

```
tracert 209.165.200.226
```

> ✅ **Respuesta:**  
> **Sí, la ruta difiere** respecto a la utilizada antes de configurar HSRP.  
> Ahora el tráfico de PC-B pasa por **R1** (router activo HSRP) en lugar de R3. La ruta es:  
> **R1 → R2 → I-Net**  
> Esto confirma que HSRP funciona correctamente: aunque PC-B pertenece a LAN 2, su tráfico sale por R1 porque es el router activo del grupo con mayor prioridad.

---

#### Paso 2 – Simular la falla del router activo (R1)

**Procedimiento:**
1. Eliminar el cable que conecta **R1** a **S1** usando la herramienta de eliminación de Packet Tracer.
2. Inmediatamente volver a **PC-B** y ejecutar: `tracert 209.165.200.226`

> ✅ **Respuesta:**  
> Al principio, el `tracert` muestra **tiempos de espera agotados** (`* * *`) mientras HSRP ejecuta su proceso de convergencia para determinar cuál router debe asumir el control.  
> Una vez que el proceso finaliza, **R3 asume el rol de router activo** y la ruta pasa a ser:  
> **R3 → R2 → I-Net**  
> La diferencia principal es que este trace tarda más en completarse (tiempo de convergencia HSRP) y usa **R3** como primer salto, en lugar de R1 como ocurría antes.

---

#### Paso 3 – Restaurar el enlace a R1

**Procedimiento:**
1. Reconectar **R1** a **S1** usando un cable **Copper Straight-Through**.
2. Ejecutar nuevamente `tracert 209.165.200.226` desde PC-B.

> ✅ **Respuestas:**
>
> **¿Qué ruta se utiliza para llegar al servidor web?**  
> Al inicio el trace puede fallar brevemente durante la reconvergencia. Luego, la ruta vuelve a pasar por **R1 → R2 → I-Net**, ya que R1 recupera automáticamente el rol de router activo gracias al comando `preempt`.
>
> **Si el comando `preempt` no se hubiera configurado en R1, ¿los resultados habrían sido los mismos?**  
> **No**. Sin `preempt`, R1 volvería a conectarse pero **no recuperaría automáticamente** el rol de router activo. R3 continuaría siendo el router activo a pesar de tener menor prioridad (100 vs 150). El tráfico seguiría pasando por R3. El comando `preempt` es precisamente lo que garantiza que el router de mayor prioridad siempre retome el control cuando vuelva a estar disponible.

---

## 📝 Configuraciones Finales Completas

### Router R1
```
enable
configure terminal
interface g0/1
 standby version 2
 standby 1 ip 192.168.1.254
 standby 1 priority 150
 standby 1 preempt
end
```
![R1 Brief](/Segundo-Corte/Laboratorio-1/capturas/R1_Brief.png)

### Router R3
```
enable
configure terminal
interface g0/0
 standby version 2
 standby 1 ip 192.168.1.254
end
```
![R1 Brief](/Segundo-Corte/Laboratorio-1/capturas/R3_Brief.png)

### Switch S1
```
enable
configure terminal
ip default-gateway 192.168.1.254
end
```

### Switch S3
```
enable
configure terminal
ip default-gateway 192.168.1.254
end
```

### PC-A y PC-B
Cambiar la puerta de enlace predeterminada a: **`192.168.1.254`**

---

## ✅ Resumen de Respuestas

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | Ruta desde PC-A al servidor web (antes de HSRP) | **R1 → R2 → I-Net** |
| 2 | Ruta desde PC-B al servidor web (antes de HSRP) | **R3 → R2 → I-Net** |
| 3 | ¿Qué ocurre cuando R3 no está disponible (sin HSRP)? | PC-B pierde toda conectividad; el tracert no encuentra ruta (`* * *`) |
| 4 | Prioridad HSRP de R3 al unirse al grupo 1 | **100** (valor por defecto) |
| 5 | ¿Cuál es el router activo? | **R1** |
| 6 | Dirección MAC de la IP virtual | **0000.0C9F.F001** |
| 7 | IP y prioridad del router en espera | IP: **192.168.1.3** — Prioridad: **100** |
| 8 | Gateway que deben usar los hosts tras configurar HSRP | **192.168.1.254** |
| 9 | ¿Los pings son exitosos tras el cambio de gateway? | **Sí** |
| 10 | ¿La ruta de PC-B cambia tras configurar HSRP? | **Sí**, ahora pasa por R1 en vez de R3 |
| 11 | ¿Cómo difiere el trace al desconectar R1? | Tiempo de espera inicial → luego ruta por **R3 → R2 → I-Net** |
| 12 | Ruta al restaurar R1 | Vuelve a pasar por **R1 → R2 → I-Net** (gracias a `preempt`) |
| 13 | ¿Mismo resultado sin `preempt` configurado en R1? | **No**, R1 no recuperaría el rol activo automáticamente |