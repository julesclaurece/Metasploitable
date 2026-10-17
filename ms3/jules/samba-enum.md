# Write-up — Samba : énumération sans authentification (Metasploitable 3, port 139/445)

Cible : `172.28.128.3` · Pas de CVE — étape d'énumération pure, pas d'exploitation.
Résultat : 2 comptes système + 3 partages + politique de mot de passe, sans identifiants.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Lister les partages (session anonyme)

```bash
smbclient -L //172.28.128.3/ -N
```
```
	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	public          Disk      WWW
	IPC$            IPC       IPC Service (metasploitable3-ub1404 server (Samba, Ubuntu))
```
Trois partages visibles **sans authentification** (`-N` = pas de mot de passe). Le commentaire
du service IPC$ révèle au passage le hostname exact : `metasploitable3-ub1404`.

Tentative de connexion au partage `public` (son commentaire "WWW" suggère un lien avec le
serveur web) :
```bash
smbclient //172.28.128.3/public -N -c "ls"
```
```
tree connect failed: NT_STATUS_ACCESS_DENIED
```
→ Visible dans la liste, mais **pas accessible en anonyme** — confirmé aussi par
`enum4linux` ensuite (`Mapping: DENIED`).

## 2. Énumération complète (`enum4linux`)

```bash
enum4linux -a 172.28.128.3
```

Résultat clé — la session anonyme est acceptée par le serveur :
```
[+] Server 172.28.128.3 allows sessions using username '', password ''
```

Comptes systèmes trouvés (énumération RID + requête utilisateurs) :
```
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: chewbacca	Name: 	Desc:

S-1-5-21-1500313315-3593119171-3274009291-501  METASPLOITABLE3-UB1404\nobody   (Local User)
S-1-5-21-1500313315-3593119171-3274009291-1000 METASPLOITABLE3-UB1404\chewbacca (Local User)
```

Politique de mot de passe (récupérée via `rpcclient`, malgré l'échec de `polenum` sur cette
install Python) :
```
Password Complexity: Disabled
Minimum Password Length: 5
```

## 3. Ce que ça donne pour la suite

- **Compte réel confirmé : `chewbacca`** (RID 1000, donc premier compte utilisateur créé
  après les comptes système par défaut — généralement un indice que c'est un compte ajouté
  intentionnellement pour le labo, pas un compte système générique).
- **Politique de mot de passe faible** (complexité désactivée, 5 caractères minimum) — une
  piste de bruteforce réaliste si on cible ce compte sur SSH/Samba/autre service partageant
  les mêmes identifiants système.
- **Trois partages identifiés**, dont `public` (lien probable avec le service web) —
  inaccessible en anonyme, donc pas une porte d'entrée directe, mais à retester une fois des
  identifiants valides trouvés.
- Aucune CVE à chercher ici : c'est de la reconnaissance passive sur un service mal
  configuré (session anonyme autorisée), pas une faille de code.

---

**Pour rejouer tout le scénario depuis zéro** : `./start.sh` (démarre la VM), puis les deux
commandes de l'étape 1-2 dans l'ordre — rien de propre à l'état de la cible à préparer au
préalable (pure lecture, aucune modification côté serveur).

**Suite du parcours MS3 :** Samba — exploitation (port 445), à partir de la version exacte
(`Samba smbd 3.X-4.X`) trouvée en reconnaissance initiale.
