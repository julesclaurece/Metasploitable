# Captures — SMTP Postfix (énumération VRFY, port 25)

Consignes de capture pour `services/smtp.html` (à construire). Commandes identiques à
`ms1/josa/smtp.md`.

> IP de ce poste (josa, réseau `vboxnet0`) : cible `192.168.56.102`.
> (Le coéquipier est sur `172.28.128.x` — ses captures à lui utiliseront ses propres IP.)

## 01-recon-nmap.png

Les deux commandes à la suite :
```bash
nmap -sC -sV -p 25 192.168.56.102
nmap --script smtp-enum-users -p 25 192.168.56.102
```

Encadré en rouge : la ligne `smtp-commands: ... VRFY ...` (1er scan) et
`Method RCPT returned a unhandled status code.` (2e scan, l'échec de la méthode par défaut).

## 02-vrfy-manuel.png

Les deux commandes à la suite (compte valide vs inventé) :
```bash
printf 'VRFY root\r\nQUIT\r\n' | nc -w3 192.168.56.102 25
printf 'VRFY nonexistentuser123\r\nQUIT\r\n' | nc -w3 192.168.56.102 25
```

Encadré en rouge : `252 2.0.0 root` et `550 5.1.1 <nonexistentuser123>: Recipient address
rejected...` — le contraste valide/invalide qui fait tout l'intérêt de la technique.

## 03-enum-lot.png

Le script de boucle (zsh) et son résultat :
```bash
USERS=(games nobody bind proxy syslog user www-data root news postgres bin mail distccd
       proftpd dhcp daemon sshd man lp mysql gnats libuuid backup msfadmin telnetd sys klog
       postfix service list irc ftp tomcat55 sync uucp)
for u in "${USERS[@]}"; do
  resp=$(printf 'VRFY %s\r\nQUIT\r\n' "$u" | nc -w2 192.168.56.102 25)
  echo "$resp" | grep -qE "^25[02] " && echo "$u : valide"
done
echo "Comptes confirmés valides : 35 / 35"
```

Encadré en rouge : la ligne finale `Comptes confirmés valides : 35 / 35` — le résultat fort de
cette étape (recoupement complet avec `samba-enum.md`).
