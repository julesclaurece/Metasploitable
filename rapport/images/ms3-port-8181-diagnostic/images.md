# Captures — Diagnostic port 8181 (MS3)

Consignes de capture pour `services/ms3-port-8181-diagnostic.html`. Commandes identiques à
celles du write-up source : `ms3/jules/port-8181-diagnostic.md`.

## 01-rescan-periodique.png

```bash
for i in 1 2 3 4 5; do nmap -p 8181 172.28.128.3 | grep 8181; sleep 20; done
```

Encadrer en rouge : les 5 lignes identiques `8181/tcp closed` — pas d'alternance.

## 02-aucune-trace.png

Via le shell déjà obtenu sur la cible (backdoor UnrealIRCd) :
```bash
ss -tlnp | grep 8181
ps aux | grep -iE "8181"
grep -rl "8181" /etc/init/ 2>/dev/null
ls /opt/
```

Encadrer en rouge : l'absence totale de résultat sur les trois premières commandes, et le
listing de `/opt/` ne contenant aucun dossier lié à 8181.
