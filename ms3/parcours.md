# Parcours d'entraînement — Metasploitable 3

Cible : **172.28.128.3** (réseau privé, isolé — ne jamais appliquer ces techniques hors de ce
labo sans autorisation écrite).

```bash
./start.sh   # démarrer
./stop.sh    # éteindre
```
SSH : `vagrant` / `vagrant` (si configuré) ou console directe.

> Ce parcours ne couvre que la VM Linux `ub1404`. Le Vagrantfile officiel de MS3 définit
> aussi une VM Windows (`win2k8`, réseau en DHCP) qu'on n'a jamais démarrée dans ce labo —
> l'essentiel de la surface "entreprise" (ManageEngine, Jenkins, Elasticsearch, Glassfish...)
> documentée dans les writeups publics de MS3 est probablement côté Windows, pas ici. À
> évaluer séparément si on veut l'ajouter au labo.

---

## Règle du jeu (pour que ce soit formateur)

- **Un service par session.** Note tout dans un carnet de bord (commandes, résultats, hypothèses).
- Pour chaque service : d'abord `nmap -sC -sV -p <port> 172.28.128.3`, puis **réfléchis avant** de chercher un exploit.
- Essaie au moins **une attaque à la main** (sans Metasploit) par service.
- Bloque-toi ~1 h sur un service avant de regarder un writeup.

Méthodologie générale : **reconnaissance → énumération → exploitation → post-exploitation.**

> Comme pour MS1, seulement **13 services réels** confirmés sur cette VM Linux (scan `-p-`
> complet + recoupement de plusieurs scans, certains ports étant intermittents).

---

## Services ouverts (recoupement de plusieurs scans `-p- -sV`, 2026-10-13)

| Port  | Service  | Version |
|-------|----------|---------|
| 21    | ftp      | ProFTPD 1.3.5 |
| 22    | ssh      | OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 |
| 80    | http     | Apache httpd 2.4.7 |
| 111   | rpcbind  | (vu de façon intermittente) |
| 139   | netbios-ssn | (vu de façon intermittente) |
| 445   | smb      | Samba smbd 3.X–4.X (workgroup WORKGROUP) |
| 631   | ipp      | CUPS 1.7 |
| 3000  | ?        | **Fermé de façon répétée** — service provisionné mais probablement crash-loopé |
| 3306  | mysql    | MySQL (banner "unauthorized") |
| 3500  | http     | WEBrick httpd 1.3.1 (Ruby 2.3.8) — appli Ruby custom |
| 6697  | irc      | **UnrealIRCd** (SSL) |
| 8080  | http     | Jetty 8.1.7.v20120910 |
| 8181  | ?        | **Fermé de façon répétée** — service provisionné mais probablement crash-loopé |

---

## 1. 🟢 FTP — ProFTPD 1.3.5 (port 21) · *Facile*

**Compétence :** relier une version à une CVE connue, exploit de bout en bout.
CVE-2015-3306 (`mod_copy`). **Fait, voir `ftp.md`.**

## 2. 🟢 Samba — énumération (port 139/445) · *Facile à moyen*

**Compétence :** l'énumération, le cœur du métier — à faire **avant** toute tentative
d'exploit.

```bash
enum4linux -a 172.28.128.3
smbclient -L //172.28.128.3/ -N
```
Partages, utilisateurs, politique de mots de passe. **Fait, voir `samba-enum.md`.**

## 3. 🟡 Samba — exploitation (port 445) · *Moyen*

**Compétence :** exploiter la version trouvée à l'étape 2, pas avant.

`searchsploit samba` pour la version exacte (3.X–4.X annoncé par nmap, à affiner).
**Fait (bloqué, cause diagnostiquée), voir `samba-exploit.md`.**

## 4. 🟡 Web — Apache (port 80) · *Moyen*

**Compétence :** la démarche web, le domaine le plus utile en vrai.

```bash
gobuster dir -u http://172.28.128.3 -w /usr/share/wordlists/dirb/common.txt
```
Énumère les répertoires, explore les applis hébergées, cherche les failles classiques
(injections, upload, identifiants faibles). Burp Suite pour intercepter/modifier.
**Fait, voir `drupal.md` (bloqué, cause diagnostiquée), `payroll-sqli.md` (succès) et
`chat-xss.md` (succès).**

## 5. 🟡 MySQL (port 3306) · *Moyen à fort*

**Compétence :** exploitation d'une base de données et le rebond.

Banner "unauthorized" — tester identifiants faibles quand même, explorer les bases
accessibles, voir comment une base compromise peut mener à l'accès système.
**Fait (bloqué, cause diagnostiquée), voir `mysql-phpmyadmin.md`.**

## 6. 🔴 Jetty — Java (port 8080) · *Fort*

**Compétence :** chaîne d'exploitation complète + post-exploitation.

Interface d'administration Java. Déployer une charge via la console, obtenir un shell, puis
**élever les privilèges** jusqu'à root. **Fait (root direct), voir `jetty-continuum.md`.**

## 7. 🟡 UnrealIRCd (port 6697) · *Trivial à facile*

**Compétence :** reconnaître une backdoor insérée dans un tarball officiel compromis — même
famille que MS2, mais ici en SSL (6697 au lieu de 6667).

CVE-2010-2075 : commande précédée de `AB;` exécutée côté serveur. Vérifier si le wrapper SSL
change la méthode de connexion du module Metasploit (`ssl` à activer dans les options).
**Fait (exécution manuelle confirmée, module Metasploit en échec diagnostiqué), voir
`unrealircd.md`.**

## 8. 🟡 WEBrick Ruby custom (port 3500) · *Moyen*

**Compétence :** auditer une appli maison plutôt qu'un produit connu — pas de CVE toute
faite à chercher sur `searchsploit`.

```bash
curl -v http://172.28.128.3:3500/
```
Explorer manuellement, identifier le framework (Sinatra/Rails léger ?), chercher des routes
non protégées, de la désérialisation Ruby (`Marshal.load`) si l'appli en fait.
**Fait (CVE-2016-0752 confirmée, bloqué sur un problème de permissions distinct de la
faille), voir `webrick-rails-render.md`.**

## 9. 🟢 CUPS (port 631) · *Facile*

**Compétence :** service souvent oublié en pentest — interface d'admin d'imprimante
accessible en HTTP.

```bash
curl http://172.28.128.3:631/
```
`searchsploit cups 1.7` — vérifier les CVE connues de cette version (exécution de code via
filtres de conversion, ou divulgation d'info via l'interface web).
**Fait (bloqué, cause diagnostiquée), voir `cups.md`.**

## 10. 🟢 rpcbind — énumération (port 111) · *Facile*

**Compétence :** énumération RPC, prérequis pour comprendre tout service RPC-dépendant.

```bash
rpcinfo -p 172.28.128.3
```
**Fait (service actif confirmé, filtrage réseau diagnostiqué par élimination), voir
`rpcbind.md`.**

## 11. 🟡 Services intermittents — port 3000 · *Diagnostic*

**Compétence :** distinguer un port vraiment fermé d'un service qui crash-loope — on vient
de rencontrer exactement ce cas en construisant ce parcours (scans incohérents entre deux
passages).

```bash
for i in 1 2 3 4 5; do nmap -p 3000 172.28.128.3 | grep 3000; sleep 20; done
```
Si le port alterne open/closed sur plusieurs scans espacés, c'est un service qui démarre et
crashe en boucle — chercher pourquoi (logs si accès shell déjà obtenu par ailleurs) plutôt
que de conclure "fermé = rien à voir". **Fait (service vivant, lié à 127.0.0.1 uniquement —
ni fermé ni crash-loopé), voir `port-3000-diagnostic.md`.**

## 12. 🟡 Services intermittents — port 8181 · *Diagnostic*

**Compétence :** même exercice que l'étape 11, deuxième occurrence pour ancrer le réflexe.

Même méthode de rescan périodique qu'à l'étape 11. **Fait (aucun service provisionné),
voir `port-8181-diagnostic.md`.**

---

**Parcours MS3 complet** — les 12 services listés ci-dessus ont tous été traités (succès,
blocage diagnostiqué, ou absence confirmée). Write-ups dans `ms3/jules/`, pages de rapport
correspondantes dans `rapport/services/ms3-*.html`.

### ⚠️ Services où l'exploitation n'a PAS abouti (bloqués, cause diagnostiquée)

Pour une vue rapide sans avoir à relire chaque section : ces services ont une vulnérabilité
**confirmée** (CVE identifiée, point d'entrée vulnérable atteint) mais **aucun accès final
obtenu** — chaque blocage est documenté avec sa cause exacte, pas laissé comme un simple
échec constaté.

- **Samba — exploitation** (item 3, port 445) — CVE-2017-7494 confirmée vulnérable, mais pas
  de partage inscriptible anonyme sur cette install. `samba-exploit.md`.
- **Drupal 7.5** (item 4, port 80) — CVE-2018-7600 confirmée, point d'entrée atteint, mais le
  cache de formulaire ne survit pas à l'injection sur cette install. `drupal.md`.
- **MySQL / phpMyAdmin** (item 5, port 3306/80) — root sans mot de passe confirmé, mais
  bloqué par deux protections indépendantes (ACL réseau MySQL + `AllowNoPassword` côté
  phpMyAdmin). `mysql-phpmyadmin.md`.
- **WEBrick / Rails Dynamic Render** (item 8, port 3500) — CVE-2016-0752 confirmée,
  technique de traversée reproduite jusqu'au rendu du fichier contrôlé, mais un problème de
  permissions filesystem (sans rapport avec la faille) empêche l'exécution du code injecté.
  `webrick-rails-render.md`.
- **CUPS 1.7.2** (item 9, port 631) — aucun vecteur non authentifié exploitable trouvé ; les
  CVE applicables exigent des identifiants valides ou un exploit natif jugé disproportionné
  pour ce point du parcours. `cups.md`.

Les items 10 à 12 (rpcbind, ports 3000/8181) sont des exercices de **diagnostic réseau**,
pas des tentatives d'exploitation — ils ne comptent pas comme des échecs : chacun conclut
avec une cause identifiée avec certitude (service mal exposé, absent, ou filtré).

---

## Ressources complémentaires

- **PortSwigger Web Security Academy** — gratuit, excellent pour le web
- **TryHackMe** / **HackTheBox Academy** — parcours guidés
- `searchsploit <produit>` — recherche de CVE hors ligne
- Liste officielle des vulnérabilités : `README.md` du dépôt metasploitable3 (couvre aussi
  la VM Windows non utilisée dans ce labo)
