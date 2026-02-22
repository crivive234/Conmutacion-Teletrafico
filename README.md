# Docker-fisrt-project
Instalación y Configuración de Docker en Arch Linux
# Instalar Docker
sudo pacman -S docker

# Iniciar y habilitar el servicio
sudo systemctl enable --now docker

# Agregar tu usuario al grupo docker (evita usar sudo en cada comando)
sudo usermod -aG docker $USER

# Cerrar sesión y volver a entrar, o ejecutar:
newgrp docker

# Verificar instalación
docker --version
docker run hello-world
