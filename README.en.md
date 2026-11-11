# SMB shares in Proxmox LXC (case study: Plex)

[🇪🇸 Español](README.md) · 🇬🇧 English

Guide to mount an **SMB/CIFS** share from a NAS on the **Proxmox host (PVE)** and expose it to one or more (unprivileged) **LXC containers** through a *bind mount*. The practical example is a **Plex** LXC.

## How it works

1. The PVE mounts the NAS SMB share at `/mnt/lxc_shares/nas`.
2. Files are mounted with the UID/GID matching those inside the LXC: in an **unprivileged** container, container IDs are shifted by 100000 on the host. So `uid=100000` is the LXC's `root` and `gid=110000` is the `lxc_shares` group (GID **10000** inside the LXC).
3. That directory is added to the LXC as a mount point (`mp0`) and the service user (Plex) gets access by joining the `lxc_shares` group.

> ℹ️ The `100000`/`110000` values assume the default mapping of an unprivileged LXC. A privileged LXC has no offset, so use the container IDs directly.

## 1. Create the group inside the LXC

Open the console of the LXC you want to give access to:

```bash
groupadd -g 10000 lxc_shares
```

## 2. Add the service user to the group (e.g. Plex)

Replace `USERNAME` with the user the service runs as. For Plex this is usually `plex` (lowercase; check with `getent passwd plex`).

```bash
usermod -aG lxc_shares USERNAME
```

Restart the service so the new group takes effect (for Plex: `systemctl restart plexmediaserver`).

## 3. Create the mount folder on the PVE

In the **Proxmox host** shell:

```bash
mkdir -p /mnt/lxc_shares/nas
```

> If `mount` complains about a missing CIFS helper, install `cifs-utils` on the PVE (`apt install cifs-utils`).

## 4. Store the SMB credentials (recommended)

`/etc/fstab` is readable by any user on the host, so it's better not to put the password there. Create a credentials file readable only by root (with your SMB user and password):

```bash
printf 'username=%s\npassword=%s\n' 'SMB_User' 'SMB_Password' > /root/.smbcredentials
chmod 600 /root/.smbcredentials
```

## 5. Add the mount to the PVE's `/etc/fstab`

Change `NAS_IP_Address` and `Share_name` to the NAS IP (or hostname) and the SMB share name:

```bash
{ echo '' ; echo '# Mount CIFS share on demand with read, write and execute permissions for LXC use' ; echo '//NAS_IP_Address/Share_name /mnt/lxc_shares/nas cifs _netdev,x-systemd.automount,noatime,uid=100000,gid=110000,dir_mode=0770,file_mode=0770,credentials=/root/.smbcredentials 0 0' ; } | tee -a /etc/fstab
```

If the share name contains spaces, write them as `\040` in `fstab`. (Less secure alternative: put `user=...,pass=...` in the options instead of `credentials=`.)

## 6. Mount the share on the PVE

```bash
systemctl daemon-reload
mount /mnt/lxc_shares/nas
ls -ln /mnt/lxc_shares/nas   # files should show uid 100000 and gid 110000
```

## 7. Add the mount point to the LXC

Replace `LXC_ID` with the container ID (run it on the PVE). Use a free `mpN` index: if the LXC already has `mp0`, use `mp1`, and so on.

```bash
pct set LXC_ID -mp0 /mnt/lxc_shares/nas,mp=/mnt/nas
```

Read-only:

```bash
pct set LXC_ID -mp0 /mnt/lxc_shares/nas,mp=/mnt/nas,ro=1
```

Manual equivalent: the line `mp0: /mnt/lxc_shares/nas,mp=/mnt/nas` in `/etc/pve/lxc/LXC_ID.conf`. `pct set` is preferable: if the container has *snapshots*, appending lines to the end of the file would place them inside the last snapshot's section instead of the active configuration.

Restart the LXC and check access:

```bash
pct reboot LXC_ID
pct exec LXC_ID -- ls -la /mnt/nas
```

## 8. Use it in Plex

In the Plex web UI, when creating or editing a library, pick the `/mnt/nas/...` folder.

## Notes

- Bind mounts are not included in `vzdump` backups.
- With unprivileged containers, permissions are the most common cause of problems: check `id USERNAME` inside the LXC and `ls -ln` on the PVE.
- Use a directory reserved for this purpose as the source (here `/mnt/lxc_shares/...`) and never bind mount system directories (`/`, `/etc`, `/var`) into a container.

## References

- [Proxmox VE — Linux Container (bind mount points)](https://pve.proxmox.com/wiki/Linux_Container)
- Thanks to [TheHellSite](https://forum.proxmox.com/members/thehellsite.88343/) on the Proxmox forum for the original tips.
