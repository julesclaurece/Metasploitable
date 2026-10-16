# Captures — Samba 3.0.20 énumération (ports 139/445)

Consignes de capture pour `services/samba-enum.html` (à construire). Commandes identiques à
`ms1/josa/samba-enum.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

## 01-recon-nmap.png

Commande :
```bash
nmap -sC -sV -p 139,445 192.168.56.102
```

Encadré en rouge : `445/tcp open netbios-ssn Samba smbd 3.0.20-Debian` et
`account_used: guest` (preuve que le script a pu se connecter en anonyme tout seul).

## 02-smbclient-shares.png

Commande :
```bash
smbclient -L //192.168.56.102/ -N
```

Encadré en rouge : `Anonymous login successful` et la liste des partages (`tmp`, `opt`).

## 03-enum4linux-users.png

Commande :
```bash
enum4linux -a 192.168.56.102
```

Capturer la section "Users on ... via RID cycling". Vu le nombre de lignes (35 comptes), ne
pas tout capturer : cadrer sur les 10-15 premières lignes de la liste, suffisant pour montrer
le principe. Encadré en rouge : les lignes `user:[root]`, `user:[msfadmin]`, `user:[service]`
— les comptes qui recoupent les identifiants documentés du labo.
