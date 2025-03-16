Esta guia sirve para saber como conectar carpetas compartidas, basadas en SMB, desde un NAS a LXC (contenedores) basados en proxmox, enseñando el caso práctico de una LXC de Plex.

# Crear el gurpo de usuarios en las LXC:

Se tiene que entrar dentro de la consola del LXC al cual se desea añadir el acceso a la carpeta compartida.

```
groupadd -g 10000 lxc_shares
```

# Añadir otros usuarios al grupo (e.g., Plex):

Si se tiene un LXC al cual se le quiere dar acceso a otras carpetas como puede ser Plex, que utiliza el usuario "Plex", habría que ejecutar el siguiente comando para poder tener acceso a los archivos de la carpeta compartida sustituyendo username por el usuario que se quiere añadir. En el caso de Plex, habría que añadir el usuario "Plex".

```
usermod -aG lxc_shares USERNAME
```

# Crear una carpeta para montar el NAS:

Para crear una carpeta donde montar la carpeta compartida, se tendria que ejecutar el siguiente comando en el shell del PVE de promox al que se le quiere dar acceso.

```
mkdir -p /mnt/lxc_shares/nas
```

# Añadir el montaje de la carpeta NAS en el PVE:

Para añadir en el archivo /etc/fstab la configuracion de montaje de la carpeta compartida, se tendra que ejecutar el siguiente comando modificando los siguientes valores:

  - Reemplazar "Dirección_IP_NAS" por la direccion a la que se quiere compartir el NAS y modificar "Carpeta_a_compartir" por la carpeta SMB.
  - Reemplazar el nombre de usuario SMB y la contraseña.

```
{ echo '' ; echo '# Montar CIFS compartido a demanda con permisos de lectura, escritura y ejecucion para el uso en LXCs' ; echo '//Dirección_IP_NAS/Carpeta_a_compartir/ /mnt/lxc_shares/nas cifs _netdev,x-systemd.automount,noatime,uid=100000,gid=110000,dir_mode=0770,file_mode=0770,user=Usuario_SMB,pass=Contraseña_SMB 0 0' ; } | tee -a /etc/fstab

```

# Montar la carpeta compartida en el PVE:

Para montar la carpeta compartida habria que realizar los siguientes dos pasos:

  1 - Recargar la configuracion que se ha actualizado en el fstab:
  
```
systemctl daemon-reload
```

  2 - Montar la carpeta compartida con el siguiente comando:
  
```
mount /mnt/lxc_shares/nas
```

# Añadir las carpetas compartidas para que se carguen en el arranque de las LXC:

Para configurar que se monte las carpetas compartidas en el arranque de las LXC ejecutar el siguiente comando cambiando el ID de las LXC

```
{ echo 'mp0: /mnt/lxc_shares/nas/,mp=/mnt/nas' ; } | tee -a /etc/pve/lxc/LXC_ID.conf
```
Si se quiere dar solo permisos de lectura, ejecutar el siguiente comando
```
{ echo 'mp0: /mnt/lxc_shares/nas/,mp=/mnt/nas,ro=1' ; } | tee -a /etc/pve/lxc/LXC_ID.conf
```
Despues de ejecutar el comando, reiniciar las LXC y acceder.

Una vez realizados estos pasos se tendría que tener acceso ya a la carpeta /mnt/nas/ desde la web de plex en la configuración de las librerias.

