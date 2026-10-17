# Captures — Diagnostic port 3000 (MS3)

Consignes de capture pour `services/ms3-port-3000-diagnostic.html`. Commandes identiques à
celles du write-up source : `ms3/jules/port-3000-diagnostic.md`.

## 01-rescan-periodique.png

```bash
for i in 1 2 3 4 5; do nmap -p 3000 172.28.128.3 | grep 3000; sleep 20; done
```

Encadrer en rouge : les 5 lignes identiques `3000/tcp closed` — pas d'alternance.

## 02-diagnostic-shell.png

Via le shell déjà obtenu sur la cible (backdoor UnrealIRCd) :
```bash
cat /etc/init/sinatra.conf
status sinatra
cd /opt/sinatra && ./server &
ss -tlnp | grep 3000
```

Encadrer en rouge : `LISTEN 0 128 127.0.0.1:3000` — le service écoute uniquement sur
loopback.
