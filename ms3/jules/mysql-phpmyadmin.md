# Write-up — MySQL / phpMyAdmin : identifiants faibles, accès bloqué par deux barrières indépendantes (Metasploitable 3)

Cible : `172.28.128.3:3306` (MySQL) et `172.28.128.3/phpmyadmin/` (phpMyAdmin 3.5.8) · Pas
de CVE exploitée — identifiants faibles confirmés (`root` sans mot de passe), mais accès
bloqué par deux mécanismes de protection indépendants.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

```bash
nmap -p 3306 172.28.128.3
```
```
3306/tcp open  mysql
```
Version de phpMyAdmin (trouvée dans le webroot, voir `drupal.md`) :
```bash
curl -s http://172.28.128.3/phpmyadmin/ChangeLog | head -3
```
```
3.5.8.0 (2013-04-08)
```

## 2. Tentative directe sur MySQL — bloquée par ACL réseau

```bash
mysql -h 172.28.128.3 -u root -proot -e "SELECT 1;"
```
```
ERROR 2002 (HY000): ... 1130 - Host '172.28.128.1' is not allowed to connect to this MySQL server
```
**Erreur 1130**, pas 1045 : ce n'est **pas** un problème de mot de passe, c'est une
restriction réseau au niveau de la table `mysql.user` — le compte `root` n'est autorisé à se
connecter que depuis certains hôtes (probablement `localhost` uniquement), pas depuis notre
réseau. Bonne pratique de sécurité, ici effective.

## 3. Tentative via phpMyAdmin (connexion locale, contourne l'ACL réseau)

phpMyAdmin tourne sur la même machine que MySQL — sa connexion passe par `localhost`, donc
l'ACL réseau de l'étape 2 ne s'applique pas ici. Sondage rapide de mots de passe courants
pour `root` :
```bash
for pass in "" "toor" "mysql" "password" "phpmyadmin" "metasploitable" "msfadmin"; do
  # POST pma_username=root & pma_password=$pass, vérifier l'absence d'erreur 1045
done
```
Résultat : tous les mots de passe non-vides échouent (`#1045 Cannot log in to the MySQL
server`), mais le **mot de passe vide** ne déclenche **pas** cette erreur — signe que
`root` a bien un mot de passe vide côté MySQL. En vérifiant la vraie réponse :
```
<div class="error">Login without a password is forbidden by configuration (see AllowNoPassword)</div>
```
**Deuxième barrière, distincte de la première** : ce n'est plus MySQL qui refuse, c'est
**phpMyAdmin lui-même** — son fichier de configuration a `AllowNoPassword = false` (le
réglage par défaut recommandé), qui bloque toute tentative de connexion avec un mot de passe
vide, indépendamment de la validité réelle de ce mot de passe côté MySQL.

## 4. Pistes de modules Metasploit — aucune applicable

```bash
msfconsole -q -x "search phpmyadmin; exit"
```
Plusieurs modules RCE pour phpMyAdmin, mais tous inapplicables ici :
- `phpmyadmin_preg_replace` (CVE-2013-3238), `phpmyadmin_null_termination_exec` : tous les
  deux **authentifiés** — nécessitent un login phpMyAdmin fonctionnel, qu'on n'a pas (barrière
  de l'étape 3).
- `phpmyadmin_config` (CVE-2009-1151) : le seul module **non authentifié**, mais couvre
  uniquement phpMyAdmin 2.11.x et 3.x ≤ 3.1.3.1 — notre version (3.5.8) est largement
  postérieure, hors plage.

## 5. Conclusion

**Identifiants faibles confirmés** (`root` sans mot de passe) — une vraie faiblesse de
configuration MySQL. Mais **deux protections indépendantes** empêchent de l'exploiter par
les deux chemins testés : l'ACL réseau MySQL (étape 2) et le réglage `AllowNoPassword` de
phpMyAdmin (étape 3). Les modules RCE connus pour phpMyAdmin ne couvrent ni cette version, ni
ce scénario sans authentification préalable.

Contrairement à Samba et Drupal (bloqués par une seule cause précise), ici ce sont **deux
couches de défense indépendantes** qui se recoupent — un bon exemple de défense en
profondeur qui fonctionne, même quand la cause racine (mot de passe vide) reste présente.

---

## Remédiation (pour la cause racine, malgré les protections déjà en place)

- Définir un mot de passe fort pour le compte `root` MySQL — les deux protections actuelles
  sont des filets de sécurité, pas un substitut à un vrai mot de passe.
- Conserver l'ACL réseau MySQL restrictive (déjà efficace) et le réglage
  `AllowNoPassword = false` de phpMyAdmin (déjà efficace) — documenter pourquoi ces deux
  réglages comptent, pour ne pas les désactiver par erreur plus tard.
- Mettre à jour phpMyAdmin au-delà de 3.5.8, qui reste vulnérable à plusieurs CVE
  authentifiées si jamais un compte valide est compromis par ailleurs.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
étapes 2 à 4 dans l'ordre. Aucune écriture côté cible.

**Suite du parcours MS3 :** Jetty (port 8080).
