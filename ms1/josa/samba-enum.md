# Write-up — Samba 3.0.20 : énumération (Metasploitable 1, ports 139/445)

Cible : `192.168.56.102` · Résultat : partages + 35 comptes système énumérés **sans
authentification**.

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`). Côté coéquipier (réseau
> `172.28.128.x`), remplacer l'IP — démarche identique.

Étape volontairement séparée de l'exploitation (`samba-exploit.md`) : l'énumération est une
compétence à part entière, à faire **avant** de chercher à exploiter quoi que ce soit.

---

## 1. Recon

🖥 Kali :
```bash
nmap -sC -sV -p 139,445 192.168.56.102
```
Sortie (extrait) :
```
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Host script results:
| smb-os-discovery:
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: metasploitable
|   Domain name: localdomain
smb-security-mode:
  account_used: guest
  message_signing: disabled (dangerous, but default)
```
→ **Samba 3.0.20-Debian**, nom de machine `metasploitable`. Le script NSE a pu se connecter en
tant que `guest` tout seul — premier signal que l'accès anonyme est permis.

## 2. Lister les partages (sans authentification)

🖥 Kali :
```bash
smbclient -L //192.168.56.102/ -N
```
Sortie :
```
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	tmp             Disk      oh noes!
	opt             Disk
	IPC$            IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
	ADMIN$          IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
```
`-N` = pas de mot de passe demandé. **`Anonymous login successful`** : le serveur accepte les
connexions sans identifiants (null session). Deux partages disque exposés : `tmp` et `opt`, en
plus des partages administratifs standards (`print$`, `IPC$`, `ADMIN$`).

## 3. Énumérer les comptes système via RID cycling

🖥 Kali :
```bash
enum4linux -a 192.168.56.102
```
Extrait de la sortie (section *Users on ... via RID cycling*) :
```
user:[root] rid:[0x3e8]
user:[msfadmin] rid:[0xbb8]
user:[user] rid:[0xbba]
user:[service] rid:[0xbbc]
user:[www-data] rid:[0x42a]
user:[tomcat55] rid:[0x4c4]
user:[postgres] rid:[0x4c0]
user:[mysql] rid:[0x4c2]
user:[proftpd] rid:[0x4ca]
...
```
**35 comptes système listés au total** (`wc -l` sur la section), dont les comptes applicatifs
déjà croisés sur d'autres services (`tomcat55`, `postgres`, `mysql`, `proftpd`...) et les
comptes utilisateurs du labo (`msfadmin`, `user`, `service` — les trois logins documentés dans
`parcours.md`).

**Mécanisme.** Le RID (*Relative Identifier*) cycling exploite le fait qu'une session SMB
*null* (sans identifiants) suffit, sur cette version de Samba mal configurée, à interroger le
SID du système (`S-1-5-21-...`) puis à énumérer séquentiellement les RID connus
(500-550, 1000-1050 par défaut dans `enum4linux`) pour retrouver le nom de compte associé à
chaque RID existant — sans jamais fournir de mot de passe.

## 4. Pourquoi c'est utile avant l'exploitation

Cette liste de comptes valides est la base directe de l'étape suivante
(`samba-exploit.md`, CVE-2007-2447) : connaître `msfadmin`/`root`/`service` comme comptes
réellement présents sur le système évite de deviner à l'aveugle, et confirme que la cible a
bien les comptes attendus du labo.

---

## Remédiation

- Désactiver l'accès anonyme/null session sur Samba (`restrict anonymous = 2` et
  `map to guest = never` dans `smb.conf`), au minimum interdire l'énumération de comptes sans
  authentification.
- Restreindre les partages exposés (`tmp`, `opt`) à des utilisateurs authentifiés et au
  principe de moindre privilège.
- Mettre à jour Samba : 3.0.20 est très ancien et vulnérable (voir `samba-exploit.md`).

## Sources

- Session réelle (ce write-up) — `nmap`, `smbclient -L`, `enum4linux -a`.

**Suite du parcours MS1 :** Samba — exploitation (port 445), CVE-2007-2447
(`usermap_script`), voir `samba-exploit.md`.
