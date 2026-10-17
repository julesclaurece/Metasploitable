# Captures — Samba énumération (MS3)

Consignes de capture pour `services/ms3-samba-enum.html`. Commandes identiques à celles du
write-up source : `ms3/jules/samba-enum.md`.

## 01-smbclient-shares.png

Les deux commandes à la suite :
```bash
smbclient -L //172.28.128.3/ -N
smbclient //172.28.128.3/public -N -c "ls"
```

Encadrer en rouge : les 3 lignes de partages (`print$`, `public`, `IPC$`) et la ligne
`NT_STATUS_ACCESS_DENIED` qui suit.

## 02-enum4linux-users.png

Commande :
```bash
enum4linux -a 172.28.128.3
```

Capturer deux extraits (la sortie complète est longue) : la ligne `allows sessions using
username '', password ''`, le compte `chewbacca` dans la section Users, et la section
Password Policy. Encadrer en rouge : `Account: chewbacca` et les 2 lignes de politique de
mot de passe.
