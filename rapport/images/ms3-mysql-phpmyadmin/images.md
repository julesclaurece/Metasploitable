# Captures — MySQL / phpMyAdmin (MS3, bloqué)

Consignes de capture pour `services/ms3-mysql-phpmyadmin.html`. Commandes identiques à
celles du write-up source : `ms3/jules/mysql-phpmyadmin.md`.

## 01-mysql-acl.png

```bash
mysql -h 172.28.128.3 -u root -proot -e "SELECT 1;"
```

Encadrer en rouge : `1130 - Host '172.28.128.1' is not allowed to connect`.

## 02-pma-password-sweep.png

Capturer le script de sondage (voir write-up) et son résultat — une ligne par mot de passe
testé, avec le résultat (échec 1045 vs pas d'erreur).

Encadrer en rouge : la ligne correspondant au mot de passe vide, seule à ne pas afficher
l'erreur 1045.

## 03-allownopassword.png

Réponse HTML complète pour la tentative avec mot de passe vide.

Encadrer en rouge : `Login without a password is forbidden by configuration (see
AllowNoPassword)`.
