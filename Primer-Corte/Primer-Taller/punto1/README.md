## Punto 1 Ejercicio a) — funbox: video ASCII en terminal

### Comando utilizado.
```bash
docker run --rm -it \
  -e XDG_RUNTIME_DIR=/tmp \
  -e DISPLAY=$DISPLAY \
  wernight/funbox bash -c \
  "cvlc --no-audio -V caca /examples/countdown.mp4 2>/dev/null"
```
### Resultado esperado
muestra un video de cuenta regresiva renderizado en caracteres ASCII
directamente en la terminal de Konsole EN Arch linux, sin necesidad de interfaz gráfica.

|Parámetro | Descripción |
|---|---|
|`--rm` | Elimina el contenedor automáticamente al terminar |
| `-it` | Modo interactivo con terminal |
| `-e XDG_RUNTIME_DIR=/tmp` | Corrige el error de variable de entorno en Wayland |
| `-e DISPLAY=$DISPLAY` | Pasa la variable de display del host al contenedor |
| `wernight/funbox` | Imagen Docker que contiene VLC y herramientas multimedia |
| `cvlc` | VLC en modo línea de comandos |
| `--no-audio` | Desactiva el audio (el contenedor no tiene PulseAudio) |
| `-V caca` | Usa el output visual de tipo ASCII art (libcaca) |
| `2>/dev/null` | Suprime mensajes de error no relevantes de VLC |
---

![Texto alternativo](punto1a.gif)

## Ejercicio b) — Docker + ROS + Gazebo: Simulación de TurtleBot3

El objetivo es simular un robot TurtleBot3 en el entorno de simulación Gazebo,
controlado por teloperación de teclado.

### Paso 1 — Descargar la imagen base de ROS
```bash
docker pull osrf/ros:noetic-desktop-full
```
### Paso 2 — Crear el archivo Dockerfile-ros

```bash
mkdir -p ~/taller-docker && cd ~/taller-docker
```
crear y agregar al archivo `Dockerfile-ros`:
```dockerfile
FROM osrf/ros:noetic-desktop-full
# Instalar dependencias del TurtleBot3
RUN apt-get update && apt-get install -y \
    ros-noetic-turtlebot3 \
    ros-noetic-turtlebot3-simulations
# Establecer el modelo del robot como variable de entorno
# Se usa ENV en lugar de ~/.bashrc porque ENV aplica al entorno
# del contenedor desde el arranque, sin depender de shells interactivos
ENV TURTLEBOT3_MODEL=burger
# Iniciar ROS y Gazebo al arrancar el contenedor
CMD ["bash", "-c", "source /opt/ros/noetic/setup.bash && roslaunch turtlebot3_gazebo turtlebot3_world.launch"]
```

> **Corrección importante:** El taller original usaba
> `RUN echo "export TURTLEBOT3_MODEL=burger" >> ~/.bashrc`, lo cual falla en arch linux porque
> `.bashrc` solo se ejecuta en shells interactivos. Se reemplazó por `ENV TURTLEBOT3_MODEL=burger`
> para que la variable esté disponible desde el inicio del contenedor.

### Paso 3 — Construir la imagen
```bash
docker build -t ros-gazebo -f Dockerfile-ros .
```

Verificar que la imagen se creó correctamente:
```bash
docker images | grep ros-gazebo
```

### Paso 4 — Ejecutar el contenedor (Terminal 1)

Primero es necesario permitir conexiones gráficas desde Docker:
```bash
xhost +local:docker
```

Luego lanzar el contenedor con soporte para Wayland/KDE:
```bash
docker run -it --rm \
  --name ros-container \
  --env="DISPLAY=$DISPLAY" \
  --env="WAYLAND_DISPLAY=$WAYLAND_DISPLAY" \
  --env="XDG_RUNTIME_DIR=/tmp" \
  --env="TURTLEBOT3_MODEL=burger" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  ros-gazebo
```
|Parámetro | Descripción |
|---|---|
| `--name ros-container` | Nombre del contenedor para referenciarlo desde otras terminales |
| `--env="DISPLAY=$DISPLAY"` | Pasa la variable de display para renderizar ventanas gráficas |
| `--env="WAYLAND_DISPLAY=$WAYLAND_DISPLAY"` | Soporte para entornos Wayland como KDE |
| `--env="XDG_RUNTIME_DIR=/tmp"` | Necesario en Wayland para evitar errores de runtime dir |
| `--env="TURTLEBOT3_MODEL=burger"` | Especifica el modelo del robot TurtleBot3 |
| `--volume="/tmp/.X11-unix:/tmp/.X11-unix:rw"` | Comparte el socket X11 para mostrar ventanas |

### Paso 5 — Operar el robot con teleoperación (Terminal 2)

Abrir una nueva pestaña en Konsole y ejecutar:
```bash
# Obtener el ID del contenedor
docker ps

# Conectarse al contenedor y lanzar la teleoperación
docker exec -it ros-container bash -c \
  "source /opt/ros/noetic/setup.bash && \
  rosrun turtlebot3_teleop turtlebot3_teleop_key"
```

Usar las teclas **W/A/S/D** para mover el robot y **X** para detenerlo o ir en reversa.

---

![Texto alternativo](punto1b.png)

## Ejercicio c) — Simulación con LIDAR y SLAM (gmapping)

El objetivo es implementar el TurtleBot3 con sensor LIDAR usando ROS y el algoritmo
SLAM gmapping para construir un mapa del entorno en tiempo real.

### Paso 1 — Crear el archivo Dockerfile-slam

Contenido del archivo `Dockerfile-slam`:
```dockerfile
FROM osrf/ros:noetic-desktop-full

# Instalar TurtleBot3, simulaciones y el paquete de SLAM gmapping
RUN apt-get update && apt-get install -y \
    ros-noetic-turtlebot3 \
    ros-noetic-turtlebot3-simulations \
    ros-noetic-slam-gmapping

# Establecer el modelo del robot como variable de entorno
ENV TURTLEBOT3_MODEL=burger

# Script de inicio: lanzar Gazebo con el mundo del TurtleBot3
CMD ["bash", "-c", "source /opt/ros/noetic/setup.bash && \
    roslaunch turtlebot3_gazebo turtlebot3_world.launch"]
```

### Paso 2 — Construir la imagen
```bash
docker build -t turtlebot3-slam -f Dockerfile-slam .
```

### Paso 3 — Ejecutar el sistema (4 terminales)

**Terminal 1 — Simulación Gazebo:**
```bash
xhost +local:docker

docker run -it --rm \
  --name slam-bot \
  --env="DISPLAY=$DISPLAY" \
  --env="WAYLAND_DISPLAY=$WAYLAND_DISPLAY" \
  --env="XDG_RUNTIME_DIR=/tmp" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  --network=host \
  turtlebot3-slam
```

**Terminal 2 — Lanzar SLAM con gmapping:**
```bash
docker exec -it slam-bot bash -c \
  "source /opt/ros/noetic/setup.bash && \
  roslaunch turtlebot3_slam turtlebot3_slam.launch slam_methods:=gmapping"
```

**Terminal 3 — Teleoperación del robot:**
```bash
docker exec -it slam-bot bash -c \
  "source /opt/ros/noetic/setup.bash && \
  rosrun turtlebot3_teleop turtlebot3_teleop_key"
```

**Terminal 4 — Visualizar el mapa en RViz:**
```bash
docker exec -it slam-bot bash -c \
  "source /opt/ros/noetic/setup.bash && \
  rosrun rviz rviz -d /opt/ros/noetic/share/turtlebot3_slam/rviz/turtlebot3_gmapping.rviz"
```

### ¿Qué hace SLAM (gmapping)?

El algoritmo `gmapping` usa los datos del sensor LIDAR del TurtleBot3 para:

1. Detectar obstáculos y paredes a su alrededor
2. Estimar la posición actual del robot dentro del entorno
3. Construir progresivamente un mapa 2D en tiempo real

A medida que se mueve el robot con la teleoperación (Terminal 3), el mapa se va
completando en RViz (Terminal 4).

---
![Texto alternativo](punto1c.png)

## Posibles mejoras

- Podemos guardar el mapa generado por SLAM usando el comando:
  ```bash
  docker exec -it slam-bot bash -c \
    "source /opt/ros/noetic/setup.bash && \
    rosrun map_server map_saver -f ~/mapa_generado"
  ```
- Usar el modelo `waffle_pi` en lugar de `burger` para simular un robot con cámara
- Implementar navegación autónoma con el paquete `move_base` una vez generado el mapa

## Trayecto Realizado

Se realizo un recorrido con el robot al rededor del area evitando los obstaculos para construir el mapa en 2D progresivamente.

[Ir al Punto 2](../punto2/README.md)
[Regresar al README Principal](../README.md)