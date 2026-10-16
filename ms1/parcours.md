# Parcours d'entraînement — Metasploitable 1

Cible : **172.28.128.4** (réseau privé isolé `vboxnet1`, même réseau que MS2/MS3 — ne jamais
appliquer ces techniques hors de ce labo sans autorisation écrite).

VM VirtualBox autonome (pas de Vagrant ici, image VMware importée telle quelle) :
```bash
./start.sh   # démarrer (console nécessaire pour le 1er login)
./stop.sh    # éteindre
```
Identifiants système (depuis le `readme.txt` officiel du build) :
- `msfadmin` / `msfadmin` · `user` / `user` · `service` / `service` · `postgres` / `postgres`

---

## Pourquoi cette VM en plus de MS2 et MS3

Metasploitable 1 (2010, Ubuntu 8.04) est bien plus ancien et "brut" que MS3 : moins de
réalisme réseau, mais des vulnérabilités **plus évidentes et plus rapides à chaîner**,
idéal pour enchaîner plusieurs sessions meterpreter dans une même séance. Bon complément :
MS3 pour la rigueur méthodologique, MS2 pour la couverture des classiques, MS1 pour le
volume de répétitions.

## Règle du jeu

Même méthodologie que MS2/MS3 : **reconnaissance (`nmap -sC -sV`) → énumération →
réflexion → tentative manuelle → exploitation → post-exploitation**, un service à la fois,
tout noté dans le carnet de bord.

> Seulement **13 ports TCP réels** sur cette VM (confirmé par scan `-p-` complet) — bien
> moins que MS2/MS3. Ce parcours descend parfois en dessous du simple numéro de port :
> plusieurs applis distinctes sur un même port (TikiWiki vs TWiki sur 80) comptent comme des
> cibles distinctes.

---

## Services ouverts (scan complet `nmap -p-` du 2026-10-13)

| Port      | Service  | Version |
|-----------|----------|---------|
| 21        | ftp      | ProFTPD 1.3.1 (Debian) |
| 22        | ssh      | OpenSSH 4.7p1 Debian-8ubuntu1 |
| 23        | telnet   | Ubuntu 8.04 |
| 25        | smtp     | Postfix |
| 53        | dns      | ISC BIND 9.4.2 |
| 80        | http     | Apache 2.2.8 + PHP 5.2.4 (TWiki, TikiWiki) |
| 137/udp   | netbios-ns | — |
| 139/445   | smb      | Unix Samba 3.0.20-Debian |
| 3306      | mysql    | 5.0.51a-3ubuntu5 |
| 3632      | distccd  | `--allow` restrictif (voir `distcc.md`) |
| 5432      | postgres | 8.3.8 |
| 8009      | ajp13    | Connecteur Tomcat (absent du `readme.txt` d'origine, trouvé en `-p-`) |
| 8180      | http     | Apache-Coyote/Tomcat 5.5 |

---

## 1. 🟢 distccd (port 3632) · *Très facile*

**Compétence :** premier RCE "gratuit", sans auth, en une commande — avec un piège d'ACL
réseau à diagnostiquer. **Fait, voir `distcc.md`.**

## 2. 🟢 Tomcat (port 8180) · *Facile*

**Compétence :** upload de WAR malveillant via une interface de manager.

`tomcat:tomcat`. Déploie un `.war` avec reverse shell JSP
(`exploit/multi/http/tomcat_mgr_deploy`, ou `msfvenom -f war` à la main).

## 3. 🟢 TikiWiki (port 80) · *Facile*

**Compétence :** exploitation d'une appli PHP vulnérable connue, premier des deux CMS
coexistant sur ce port.

`tikiwiki_graph_formula` (RCE via `tiki-graph_formula.php`, injection PHP) → shell direct.

## 4. 🟢 TWiki (port 80) · *Facile*

**Compétence :** même port, appli différente de TikiWiki — traiter comme une cible séparée
plutôt que de supposer qu'un seul scan "port 80" suffit.

Faille de divulgation d'info/RCE via `search.cgi`. Bon exercice pour lire un exploit public
et l'adapter.

## 5. 🟡 Samba — exploitation (port 445) · *Facile à moyen*

**Compétence :** exploitation d'une version Samba ancienne, après énumération (étape 6).

**CVE-2007-2447** confirmé applicable à Samba 3.0.20 (source : walkthrough externe
spécifique à MS1, recoupé). L'option `username map script` mal configurée permet d'injecter
des commandes shell via le nom d'utilisateur. `exploit/multi/samba/usermap_script`.

## 6. 🟡 Samba — énumération (port 139/445) · *Facile*

**Compétence :** l'énumération, le cœur du métier — à faire **avant** l'étape 5, pas après.

```bash
enum4linux -a 172.28.128.4
smbclient -L //172.28.128.4/ -N
```
Partages, utilisateurs, politique de mots de passe — séparé volontairement de l'exploitation
pour marquer que ce sont deux compétences distinctes.

## 7. 🟡 NetBIOS name service (port 137/udp) · *Facile*

**Compétence :** énumération au niveau nom NetBIOS, en amont de SMB.

```bash
nmblookup -A 172.28.128.4
```
Révèle le nom de machine/domaine avant même de toucher SMB — utile pour construire une
wordlist de usernames/hostnames.

## 8. 🟡 MySQL (port 3306) · *Moyen*

**Compétence :** compromission de base de données, pivot.

`root:root` (ou vide selon config). Cherche la lecture de fichiers si privilège `FILE`
accordé.

## 9. 🟡 PostgreSQL (port 5432) · *Moyen*

**Compétence :** même logique que MySQL, service séparé.

`postgres:postgres`. `COPY ... TO PROGRAM` pour RCE si la version le permet.

## 10. 🟡 AJP13 (port 8009) · *Moyen*

**Compétence :** attaquer le connecteur Tomcat directement plutôt que l'interface HTTP —
surface différente de l'étape 2.

Le protocole AJP n'est pas censé être exposé publiquement. Vérifier si cette version est
sensible à une variante de Ghostcat (CVE-2020-1938, normalement sur Tomcat plus récent — bon
exercice pour comprendre pourquoi ça ne s'applique probablement pas ici et pourquoi vérifier
quand même).

## 11. 🟢 DNS — BIND 9.4.2 (port 53) · *Facile*

**Compétence :** énumération DNS, transfert de zone.

```bash
dig axfr @172.28.128.4 metasploitable.localdomain
```

## 12. 🟢 SMTP — Postfix (port 25) · *Facile*

**Compétence :** énumération d'utilisateurs via commandes SMTP.

```bash
nmap --script smtp-enum-users -p 25 172.28.128.4
```
`VRFY`/`EXPN` pour confirmer des comptes système valides.

## 13. 🟡 ProFTPD 1.3.1 (port 21) · *Moyen*

**Compétence :** recherche de vuln sans certitude de résultat — le `readme.txt` officiel du
build note "pas de faille injectée confirmée" pour ce service.

```bash
searchsploit proftpd 1.3.1
```
Bon exercice pour accepter qu'un service **peut ne rien donner** — ne pas s'acharner, noter
l'hypothèse et passer au suivant (règle du jeu : 1h max avant de laisser tomber).

---

## Ressources complémentaires

- `readme.txt` embarqué dans l'archive VM (`~/Téléchargements/metasploitable1/Metasploitable/readme.txt`)
- g0tmi1k — writeups d'origine (distcc, tomcat, tikiwiki) au moment de la sortie de la VM
- `searchsploit <produit>` pour toute version listée ci-dessus
