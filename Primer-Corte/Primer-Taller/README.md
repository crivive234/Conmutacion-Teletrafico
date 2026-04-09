# Punto 1 — Instalando y entendiendo el alcance de Docker.

## Sistema Operativo
- **Distribucion:** Arch Linux x86_64
- **Kernel:** Linux 6.18.9-arch1-2
- **Entorno de escritorio:** KDE Plasma 6.6.0
- **Gestor de ventanas:** KWin (Wayland)

---

## Instalación de Docker en Arch Linux

### Paso 1 — Instalar Docker
```bash
sudo pacman -S docker
```

### Paso 2 — Iniciar y habilitar el servicio
```bash
sudo systemctl enable --now docker
```

### Paso 3 — Agregar el usuario al grupo docker
Esto permite usar Docker sin necesidad de `sudo` en cada comando.
```bash
sudo usermod -aG docker $USER
```

### Paso 4 — Aplicar el grupo sin reiniciar sesión
```bash
newgrp docker
```

### Paso 5 — Verificar la instalación
```bash
docker --version
docker run hello-world
```
Si `hello-world` muestra un mensaje de bienvenida, Docker está correctamente instalado.

---

## Consideraciones para KDE Plasma con Wayland

Dado que el sistema utiliza **Wayland con KWin**, los contenedores que necesitan mostrar
ventanas gráficas requieren pasos adicionales respecto a un sistema X11 estándar.

### Instalar xorg-xhost
Permite que los contenedores Docker accedan al servidor de display.
```bash
sudo pacman -S xorg-xhost
```

### Permitir conexiones locales de Docker antes de ejecutar contenedores gráficos
```bash
xhost +local:docker
```
Este comando debe ejecutarse cada vez antes de lanzar un contenedor que abra ventanas.

[Ir al Punto 1](punto1/README.md)