# SMB en LXC de Proxmox (caso práctico: Plex)

🇪🇸 Español · [🇬🇧 English](README.en.md)

Guía para montar una carpeta compartida por **SMB/CIFS** desde un NAS en el **host Proxmox (PVE)** y exponerla a uno o varios **contenedores LXC** (no privilegiados) mediante un *bind mount*. El caso práctico es un LXC de **Plex**.

## Cómo funciona

1. El PVE monta la carpeta SMB del NAS en `/mnt/lxc_shares/nas`.
2. Los archivos se montan con el UID/GID que corresponden dentro del LXC: en un contenedor **no privilegiado** los IDs del contenedor se desplazan 100000 en el host. Por eso `uid=100000` es el `root` del LXC y `gid=110000` es el grupo `lxc_shares` (GID **10000** dentro del LXC).
3. Ese directorio se añade al LXC como punto de montaje (`mp0`) y se da acceso al usuario del servicio (Plex) añadiéndolo al grupo `lxc_shares`.

> ℹ️ Los valores `100000`/`110000` asumen el mapeo por defecto de un LXC no privilegiado. En un LXC privilegiado no hay desplazamiento y debes usar directamente los IDs del contenedor.

## 1. Crear el grupo dentro del LXC

Entra en la consola del LXC al que quieres dar acceso:

```bash
groupadd -g 10000 lxc_shares
```

## 2. Añadir al grupo el usuario del servicio (p. ej. Plex)

Sustituye `USERNAME` por el usuario con el que se ejecuta el servicio. En Plex suele ser `plex` (en minúsculas; compruébalo con `getent passwd plex`).

```bash
usermod -aG lxc_shares USERNAME
```

Reinicia el servicio para que aplique el nuevo grupo (en Plex: `systemctl restart plexmediaserver`).

## 3. Crear la carpeta de montaje en el PVE

En la shell del **host Proxmox**:

```bash
mkdir -p /mnt/lxc_shares/nas
```

> Si `mount` se queja de que falta el *helper* de CIFS, instala `cifs-utils` en el PVE (`apt install cifs-utils`).

## 4. Guardar las credenciales SMB (recomendado)

`/etc/fstab` es legible por cualquier usuario del host, así que es mejor no escribir ahí la contraseña. Crea un archivo de credenciales solo para root (con tu usuario y contraseña SMB):

```bash
printf 'username=%s\npassword=%s\n' 'Usuario_SMB' 'Contraseña_SMB' > /root/.smbcredentials
chmod 600 /root/.smbcredentials
```

## 5. Añadir el montaje al `/etc/fstab` del PVE

Cambia `Dirección_IP_NAS` y `Carpeta_a_compartir` por la IP (o nombre) del NAS y el nombre del recurso SMB:

```bash
{ echo '' ; echo '# Montar CIFS compartido a demanda con permisos de lectura, escritura y ejecución para el uso en LXCs' ; echo '//Dirección_IP_NAS/Carpeta_a_compartir /mnt/lxc_shares/nas cifs _netdev,x-systemd.automount,noatime,uid=100000,gid=110000,dir_mode=0770,file_mode=0770,credentials=/root/.smbcredentials 0 0' ; } | tee -a /etc/fstab
```

Si el nombre del recurso contiene espacios, escríbelos como `\040` en `fstab`. (Alternativa menos segura: poner `user=...,pass=...` en las opciones en lugar de `credentials=`.)

## 6. Montar la carpeta en el PVE

```bash
systemctl daemon-reload
mount /mnt/lxc_shares/nas
ls -ln /mnt/lxc_shares/nas   # los archivos deben aparecer con uid 100000 y gid 110000
```

## 7. Añadir el punto de montaje al LXC

Sustituye `LXC_ID` por el ID del contenedor (ejecútalo en el PVE). Usa un índice `mpN` libre: si el LXC ya tiene `mp0`, usa `mp1`, etc.

```bash
pct set LXC_ID -mp0 /mnt/lxc_shares/nas,mp=/mnt/nas
```

Solo lectura:

```bash
pct set LXC_ID -mp0 /mnt/lxc_shares/nas,mp=/mnt/nas,ro=1
```

Equivalente manual: la línea `mp0: /mnt/lxc_shares/nas,mp=/mnt/nas` en `/etc/pve/lxc/LXC_ID.conf`. Es preferible `pct set`: si el contenedor tiene *snapshots*, añadir líneas al final del archivo las colocaría dentro de la sección del último snapshot en lugar de la configuración activa.

Reinicia el LXC y comprueba el acceso:

```bash
pct reboot LXC_ID
pct exec LXC_ID -- ls -la /mnt/nas
```

## 8. Usarlo en Plex

En la web de Plex, al crear o editar una biblioteca, elige la carpeta `/mnt/nas/...`.

## Notas

- Los *bind mounts* no entran en las copias de seguridad de `vzdump`.
- Con contenedores no privilegiados los permisos son la causa más habitual de problemas: revisa `id USERNAME` dentro del LXC y `ls -ln` en el PVE.
- Usa como origen un directorio reservado para este fin (aquí `/mnt/lxc_shares/...`) y nunca montes directorios del sistema (`/`, `/etc`, `/var`) en un contenedor.

## Referencias

- [Proxmox VE — Linux Container (bind mount points)](https://pve.proxmox.com/wiki/Linux_Container)
- Agradecimiento a [TheHellSite](https://forum.proxmox.com/members/thehellsite.88343/) en el foro de Proxmox por los consejos originales.
