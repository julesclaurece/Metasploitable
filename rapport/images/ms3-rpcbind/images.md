# Captures — rpcbind (MS3, diagnostic)

Consignes de capture pour `services/ms3-rpcbind.html`. Commandes identiques à celles du
write-up source : `ms3/jules/rpcbind.md`.

## 01-recon-filtered.png

```bash
nmap -sC -sV -p 111 172.28.128.3
nmap -sU -p 111 172.28.128.3
```

Encadrer en rouge : `filtered` (TCP) et `open|filtered` (UDP) — l'ambiguïté caractéristique
d'un port dont les paquets sont silencieusement ignorés.

## 02-diagnostic-shell.png

Via le shell déjà obtenu sur la cible (backdoor UnrealIRCd) :
```bash
ss -lunp | grep 111
ss -ltnp | grep 111
ps aux | grep -i rpc
```

Encadrer en rouge : `rpcbind` en cours d'exécution (root), lié sur `*:111` en TCP et UDP —
le service tourne correctement.
