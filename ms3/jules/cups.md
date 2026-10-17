# Write-up — CUPS 1.7.2 (Metasploitable 3, port 631)

Cible : `172.28.128.3:631` · Résultat : **bloqué** — aucun vecteur non authentifié exploitable
trouvé, les CVE pertinentes exigent soit des identifiants valides, soit un exploit natif
complexe jugé disproportionné pour ce point du parcours.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

```bash
nmap -sC -sV -p 631 172.28.128.3
```
```
631/tcp open  ipp     CUPS 1.7
|_http-title: Home - CUPS 1.7.2
| http-methods:
|_  Potentially risky methods: PUT
```
Interface web CUPS standard accessible sans authentification en lecture (page d'accueil,
imprimantes, aide) — l'administration (changement de config, ajout d'imprimante) demande une
authentification.

## 2. Méthode d'identification des CVE et des modules

```bash
searchsploit cups
```
```
CUPS 1.3.7 - 'HP-GL/2' Filter Remote Code Execution      | linux/remote/32470.rb
CUPS < 2.0.3 - Remote Command Execution                  | linux/remote/41233.py
CUPS < 1.3.8-4 - Local Privilege Escalation              | multiple/local/7550.c
```
```bash
msfconsole -q -x "search cups; exit"
```
```
post/multi/escalate/cups_root_file_read              2012-11-20  normal
exploit/multi/http/cups_bash_env_exec                2014-09-24  excellent
exploit/multi/misc/cups_ipp_remote_code_execution    2024-09-26  normal
```
Trois pistes examinées :

- **`exploit/multi/http/cups_bash_env_exec`** (Shellshock via filtre CUPS, CVE-2014-6271) :
  le module lui-même exige des options `HttpUsername`/`HttpPassword` valides — authentifié
  par design, pas un bypass.
- **`post/multi/escalate/cups_root_file_read`** (CVE-2012-5519) : cible explicitement
  CUPS **< 1.6.2** et exige que la session appartienne au groupe `lpadmin` — notre version
  (1.7.2) est postérieure au correctif, et aucune session de ce type n'est disponible pour
  ce service précis.
- **`linux/remote/41233.py`** (CVE-2015-1158, « Reference Count Over Decrement ») : cible
  CUPS **< 2.0.3** (1.7.2 est concernée), mais l'exploit repose sur une confusion de type
  dans la gestion mémoire d'IPP, nécessitant la compilation d'une bibliothèque partagée
  malveillante et une séquence précise de requêtes IPP pour gagner la condition de
  concurrence — complexité jugée disproportionnée pour ce point du parcours, non tentée.

## 3. Tentative manuelle — PUT non authentifié

`nmap` signale la méthode `PUT` comme potentiellement risquée — testée directement sur le
fichier de configuration (technique historique CVE-2008-5183, pour les versions très
anciennes de CUPS) :
```bash
curl -X PUT --data-binary @testfile.txt http://172.28.128.3:631/admin/conf/cupsd.conf
```
```
HTTP 426 Upgrade Required — redirection vers https://...
```
```bash
curl -k -X PUT --data-binary @testfile.txt https://172.28.128.3:631/admin/conf/cupsd.conf
```
```
HTTP 401 Unauthorized
```
CUPS exige HTTPS **et** une authentification valide pour toute modification de
configuration — ce vecteur est correctement bloqué sur cette version. Tentative de PUT sur
un fichier arbitraire du webroot :
```bash
curl -X PUT --data-binary @testfile.txt http://172.28.128.3:631/testupload.txt
```
```
HTTP 403 Forbidden
```
Pas d'écriture arbitraire possible non plus. Sweep rapide d'identifiants par défaut/faibles
contre l'authentification HTTP Basic (`root`, `admin`, `lp` avec mots de passe évidents) :
tous rejetés (401 systématique).

## 4. Conclusion

Service correctement identifié et versionné, CVE pertinentes recherchées et comprises, mais
**aucun vecteur non authentifié n'a été trouvé exploitable** sur cette install précise : le
chemin PUT historique est bloqué par l'exigence HTTPS + authentification, aucun identifiant
faible ne fonctionne, et les CVE applicables à cette version exigent soit des identifiants
valides (Shellshock), soit un exploit natif à la fiabilité incertaine et à la complexité
disproportionnée pour ce point du parcours (CVE-2015-1158).

---

## Remédiation

- Mettre à jour CUPS au-delà de 2.0.3 pour corriger CVE-2015-1158, même si elle n'a pas été
  exploitée ici — la présence de versions vulnérables à des CVE connues reste un risque.
- Restreindre l'accès réseau à l'interface d'administration CUPS (631/tcp) aux réseaux de
  gestion, elle n'a aucune raison d'être exposée aux clients.
- Si Shellshock (CVE-2014-6271) n'est pas corrigé au niveau du système (bash), le corriger
  indépendamment de CUPS — c'est une faille système, pas applicative.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
étapes 1 à 3 dans l'ordre.

**Suite du parcours MS3 :** rpcbind — énumération (port 111).
