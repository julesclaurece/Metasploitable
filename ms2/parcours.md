# Parcours d'entraînement — Metasploitable 2

Cible : **172.28.128.5** (réseau privé isolé `vboxnet1`, même réseau que MS1/MS3 — ne jamais
appliquer ces techniques hors de ce labo sans autorisation écrite).

VM VirtualBox autonome (pas de Vagrant ici, image VMware importée telle quelle) :
```bash
./start.sh   # démarrer (console nécessaire pour le 1er login)
./stop.sh    # éteindre
```
Identifiants console par défaut : **`msfadmin` / `msfadmin`**.

---

## Pourquoi cette VM en plus de MS1 et MS3

Metasploitable 2 (2012) est la version la plus utilisée dans l'écosystème formation
(TryHackMe, HTB Academy, innombrables writeups). Beaucoup plus riche en services que MS1 —
c'est ici que tu croises les grands classiques qu'on retrouve partout en vrai : backdoors
célèbres (vsftpd, UnrealIRCd), RPC/NFS, services "rlogin-like" en clair, VNC avec mot de passe
faible. Bon complément : MS1 pour le volume, MS3 pour le réalisme, MS2 pour la couverture des
classiques du métier.

## Règle du jeu

Même méthodologie que les autres cibles du dépôt : **reconnaissance (`nmap -sC -sV`) →
énumération → réflexion → tentative manuelle → exploitation → post-exploitation**, un service
à la fois, tout noté dans le carnet de bord.

---

## Services ouverts (scan complet `nmap -p- -sV` du 2026-10-13)

| #  | Port  | Service      | Détail |
|----|-------|--------------|--------|
| 1  | 21    | ftp          | **vsftpd 2.3.4** (backdoor) |
| 2  | 22    | ssh          | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 3  | 23    | telnet       | Linux telnetd |
| 4  | 25    | smtp         | Postfix |
| 5  | 53    | domain       | ISC BIND 9.4.2 |
| 6  | 80    | http         | Apache 2.2.8 (DVWA, Mutillidae, phpMyAdmin, TWiki...) |
| 7  | 111   | rpcbind      | RPC #100000 |
| 8  | 139/445 | netbios-ssn/smb | Samba smbd 3.X–4.X |
| 9  | 512   | exec         | netkit-rsh rexecd |
| 10 | 513   | login        | rlogind |
| 11 | 514   | shell        | netkit rshd |
| 12 | 1099  | java-rmi     | GNU Classpath grmiregistry |
| 13 | 1524  | bindshell    | **"Metasploitable root shell"** (artefact, pas un service) |
| 14 | 2049 + 53310/51427/44867 | nfs + rpc.mountd/nlockmgr/statd | Export racine `*(rw,no_root_squash)` |
| 15 | 2121  | ftp          | ProFTPD 1.3.1 (second FTP, port alternatif) |
| 16 | 3306  | mysql        | MySQL 5.0.51a-3ubuntu5 |
| 17 | 3632  | distccd      | `--allow 0.0.0.0/0` (hors top-1000, trouvé en `-p-`) |
| 18 | 5432  | postgresql   | PostgreSQL 8.3.0–8.3.7 |
| 19 | 5900  | vnc          | VNC protocole 3.3 |
| 20 | 6000  | X11          | Accès refusé (ouvert mais protégé) |
| 21 | 6667/6697 | irc/ircs-u | **UnrealIRCd** (clair + SSL) |
| 22 | 8009  | ajp13        | Apache JServ Protocol 1.3 (connecteur Tomcat) |
| 23 | 8180  | http         | Apache Tomcat/Coyote JSP engine 1.1 |
| 24 | 8787  | drb          | Distributed Ruby rogue service (RCE direct) |

Scan complet (`-p-`) nécessaire : `distcc` (3632) et `drb` (8787) n'apparaissent pas dans un
scan top-1000 par défaut.

---

## 1. 🟢 vsftpd 2.3.4 (port 21) · *Trivial*

**Compétence :** reconnaître une backdoor insérée volontairement dans un code source
(supply-chain historique, CVE-2011-2523).

Login contenant `:)` → shell root en écoute sur le port 6200. Pas d'exploit à chercher —
comprends le mécanisme, puis `exploit/unix/ftp/vsftpd_234_backdoor`. **Fait, voir
`vsftpd.md`.**

## 2. 🟢 UnrealIRCd (port 6667) · *Trivial*

**Compétence :** même famille de vulnérabilité (backdoor dans un tarball officiel compromis).

CVE-2010-2075 : toute commande précédée de `AB;` sur le port d'écoute s'exécute côté serveur.
`exploit/unix/irc/unreal_ircd_3281_backdoor`.

## 3. 🟢 Distributed Ruby / drb (port 8787) · *Trivial*

**Compétence :** service de debug laissé grand ouvert en production — RCE direct sans auth.

Un service `drb` (Distributed Ruby) tourne sans restriction : n'importe quel client Ruby peut
appeler des méthodes arbitraires côté serveur. `exploit/linux/misc/drb_remote_codeexec`.

## 4. 🟡 Samba — usermap_script (port 139/445) · *Facile*

**Compétence :** RCE sur une mauvaise configuration Samba (pas une vraie faille de code).

CVE-2007-2447 : `username map script` mal configuré permet d'injecter des commandes shell via
le nom d'utilisateur. `exploit/multi/samba/usermap_script`.

## 5. 🟡 Bindshell abandonné (port 1524) · *Trivial mais instructif*

**Compétence :** reconnaître un artefact plutôt qu'une vraie vulnérabilité — nmap l'identifie
lui-même comme `"Metasploitable root shell"`.

`nc 172.28.128.5 1524` suffit. Rappel que tout ce qui écoute n'est pas une faille à
"exploiter" — ici c'est une porte déjà ouverte par un exploit précédent (feature du lab, pas
service réel).

## 6. 🟡 distccd (port 3632) · *Facile*

**Compétence :** même RCE que sur MS1, mais sans l'obstacle ACL — comparer les deux configs.

`--allow 0.0.0.0/0` ici contre une whitelist restrictive sur MS1 — bon exercice pour voir
comment la même CVE (2004-2687) peut être bloquée ou grande ouverte selon la conf.
`exploit/unix/misc/distcc_exec`.

## 7. 🟡 Java RMI (port 1099) · *Moyen*

**Compétence :** désérialisation Java non authentifiée — famille encore très actuelle
(Log4Shell et consorts).

Le registre RMI expose une interface de classe sans authentification.
`exploit/multi/misc/java_rmi_server`.

## 8. 🟡 ProFTPD 1.3.1 (port 2121) · *Moyen*

**Compétence :** comparer à ProFTPD 1.3.5 (MS3) — version différente, vecteur différent
(pas de `mod_copy` ici).

`searchsploit proftpd 1.3.1` — chercher la CVE adaptée à cette version précise plutôt que de
réutiliser le réflexe `mod_copy` de MS3.

## 9. 🔴 NFS root export (port 2049) · *Moyen à fort*

**Compétence :** confiance excessive basée sur l'export, pas l'authentification.

```bash
showmount -e 172.28.128.5
```
Export `/` en `rw,no_root_squash` : montage possible depuis Kali, lecture/écriture directe du
filesystem cible, y compris dépôt de clé SSH dans `/root/.ssh/` ou `/home/*/.ssh/` pour un
accès root/utilisateur sans exploit.

## 10. 🔴 rexec / rlogin / rsh (ports 512-514) · *Fort*

**Compétence :** confiance basée sur l'IP source (`.rhosts`) — classe de vulnérabilité
historique, toujours pertinente conceptuellement (trust boundary).

Teste `rlogin -l root 172.28.128.5` après avoir identifié un `.rhosts` permissif, ou
bruteforce via `rsh`.

## 11. 🔴 MySQL (port 3306) · *Moyen*

**Compétence :** compromission de base de données, pivot.

`root:root` ou vide. Une fois dedans, cherche la lecture de fichiers (`LOAD_FILE`) si
`FILE` priv accordé.

## 12. 🔴 PostgreSQL (port 5432) · *Moyen*

**Compétence :** même logique que MySQL, avec `pg_hba.conf` ouvert en `0.0.0.0/0`.

`postgres:postgres`. `COPY ... TO PROGRAM` pour RCE si version le permet.

## 13. 🔴 Tomcat (port 8180) · *Fort*

**Compétence :** chaîne complète identifiants faibles → déploiement WAR → shell.

`tomcat:tomcat` (manager). Mêmes techniques que sur MS1/MS3.

## 14. 🔴 VNC (port 5900) · *Facile*

**Compétence :** mot de passe par défaut sur un accès graphique complet.

Mot de passe `password`. `vncviewer 172.28.128.5` ou module Metasploit
`auxiliary/scanner/vnc/vnc_login`. Accès bureau complet, pas juste un shell.

## 15. 🔴 X11 (port 6000) · *Moyen*

**Compétence :** exposition d'un serveur d'affichage X sans contrôle d'accès.

`xhost`/`xwd` pour capturer l'écran ou injecter des événements clavier si le contrôle d'accès
est désactivé (`xhost +`).

## 16. 🔴 Web — Apache + apps (port 80) · *Fort*

**Compétence :** la démarche web complète, plusieurs applis vulnérables empilées sur le même
port.

DVWA, Mutillidae, phpMyAdmin, TWiki — chacune mérite sa propre session d'énumération
(`gobuster`/`ffuf`) et ses propres techniques (injections SQL, upload, LFI).

## 17. 🟢 DNS — BIND 9.4.2 (port 53) · *Facile*

**Compétence :** énumération DNS, transfert de zone.

```bash
dig axfr @172.28.128.5 metasploitable.localdomain
```
Version ancienne, vérifier `searchsploit bind 9.4.2` pour DoS/RCE connus en complément de
l'énumération.

## 18. 🟢 SMTP — Postfix (port 25) · *Facile*

**Compétence :** énumération d'utilisateurs via commandes SMTP.

```bash
nmap --script smtp-enum-users -p 25 172.28.128.5
```
`VRFY`/`EXPN` pour confirmer des comptes système valides, utile pour bruteforce SSH/Telnet
ensuite.

## 19. 🟢 rpcbind (port 111) · *Facile*

**Compétence :** énumération RPC pure, prérequis pour NFS/rlogin.

```bash
rpcinfo -p 172.28.128.5
```
Révèle tous les services RPC dynamiques (mountd, nlockmgr, status) et leurs ports réels —
c'est ce qui a permis d'identifier les 4 ports hauts "unknown" du scan `-p-`.

---

## Ressources complémentaires

- `searchsploit <produit>` pour chaque version listée ci-dessus
- Les writeups Metasploitable 2 sont extrêmement nombreux en ligne — bon réflexe pour
  recouper une hypothèse, mais **essaie d'abord sans regarder** (règle du jeu du dépôt).
