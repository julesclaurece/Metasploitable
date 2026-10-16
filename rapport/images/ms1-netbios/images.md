# Captures — NetBIOS Name Service (port 137/udp)

Consignes de capture pour `services/netbios.html` (à construire). Commande identique à
`ms1/josa/netbios.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

## 01-nmblookup.png

Commande :
```bash
nmblookup -A 192.168.56.102
```

Encadré en rouge : la ligne `METASPLOITABLE <20> - B <ACTIVE>` (File Server Service, annonce
le partage de fichiers actif avant même l'énumération SMB) et `WORKGROUP <00> - <GROUP>`
(nom du domaine).
