# Captures — CUPS 1.7.2 (MS3, bloqué)

Consignes de capture pour `services/ms3-cups.html`. Commandes identiques à celles du
write-up source : `ms3/jules/cups.md`.

## 01-recon-nmap.png

```bash
nmap -sC -sV -p 631 172.28.128.3
```

Encadrer en rouge : `CUPS 1.7` et `Potentially risky methods: PUT`.

## 02-recherche-cve.png

```bash
searchsploit cups
msfconsole -q -x "search cups; exit"
```

Encadrer en rouge : les 3 pistes retenues (`cups_bash_env_exec`, `cups_root_file_read`,
`41233.py` / CVE-2015-1158).

## 03-put-bloque.png

```bash
curl -X PUT --data-binary @testfile.txt http://172.28.128.3:631/admin/conf/cupsd.conf
curl -k -X PUT --data-binary @testfile.txt https://172.28.128.3:631/admin/conf/cupsd.conf
```

Encadrer en rouge : `426 Upgrade Required` puis `401 Unauthorized` — le vecteur PUT
historique est bloqué par l'exigence HTTPS + authentification.
