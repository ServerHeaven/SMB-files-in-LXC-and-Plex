Esta guia sirve para saber como conectar carpetas compartidas, basadas en SMB, desde un NAS a LXC (contenedores) basados en proxmox, enseñando el caso práctico de una LXC de Plex.
Primero hay que crea 
# Crear el gurpo de usuarios en las LXC:
Se tiene que entrar dentro de la barra de comandos del LXC al cual se desea añadir el acceso a la carpeta compartida
```
groupadd -g 10000 lxc_shares
```

# Añadir otros usuarios al grupo (e.g., Jellyfin, Plex):
Si se tiene un LXC al cual se le quiere dar acceso a otras carpetas como puede ser Plex, que utiliza el usuario "Plex", habría que ejecutar el siguiente comando para poder tener acceso a los archivos de la carpeta compartida

```
usermod -aG lxc_shares USERNAME
```

# Crear una carpeta para montar el NAS
Para crear una carpeta donde montar la carpeta compartida, se tendria que ejecutar el siguiente comando en el shell del PVE de promox al que se le quiere dar acceso.
```
mkdir -p /mnt/lxc_shares/nas
```

# Añadir el montaje de la carpeta NAS en el PVE
Para añadir en el archivo /etc/fstab la configuracion de montaje de la carpeta compartida, se tendra que ejecutar el siguiente comando modificando los siguientes valores:
  -Reemplazar "Dirección_IP_NAS" por la direccion a la que se quiere compartir el NAS
Replace Username and Passwords
```
{ echo '' ; echo '# Montar CIFS compartido a demanda con permisos de lectura, escritura y ejecucion para el uso en LXCs' ; echo '//Dirección_IP_NAS/Carpeta_a_compartir/ /mnt/lxc_shares/nas cifs _netdev,x-systemd.automount,noatime,uid=100000,gid=110000,dir_mode=0770,file_mode=0770,user=Usuario_SMB,pass=Contraseña_SMB 0 0' ; } | tee -a /etc/fstab

```

# Mount the NAS to the Proxmox Host
```
mount /mnt/lxc_shares/nas_rwx
```

# Add Bind Mount of the Share to the LXC Config
Be sure to change the LXC_ID
```
You can mount it in the LXC with read+write+execute (rwx) permissions.
{ echo 'mp0: /mnt/lxc_shares/nas_rwx/,mp=/mnt/nas' ; } | tee -a /etc/pve/lxc/LXC_ID.conf

You can also mount it in the LXC with read-only (ro) permissions.
{ echo 'mp0: /mnt/lxc_shares/nas_rwx/,mp=/mnt/nas,ro=1' ; } | tee -a /etc/pve/lxc/LXC_ID.conf
```

Thanks to https://forum.proxmox.com/members/thehellsite.88343/ for tips
