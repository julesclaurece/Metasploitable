# Write-up — NetBIOS Name Service (Metasploitable 1, port 137/udp)

Cible : `192.168.56.102` · Résultat : nom de machine et domaine révélés, sans toucher SMB.

> **Note réseau.** Je suis sur `vboxnet0` (cible `192.168.56.102`). Côté coéquipier (réseau
> `172.28.128.x`), remplacer l'IP — démarche identique.

Étape volontairement rapide : l'objectif est de montrer qu'on peut récupérer des informations
utiles (nom de machine, domaine) **avant** d'entamer l'énumération SMB elle-même (`samba-enum.md`).

---

## 1. Interrogation NetBIOS

🖥 Kali :
```bash
nmblookup -A 192.168.56.102
```
Sortie :
```
Looking up status of 192.168.56.102
	METASPLOITABLE  <00> -         B <ACTIVE>
	METASPLOITABLE  <03> -         B <ACTIVE>
	METASPLOITABLE  <20> -         B <ACTIVE>
	..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>
	WORKGROUP       <00> - <GROUP> B <ACTIVE>
	WORKGROUP       <1d> -         B <ACTIVE>
	WORKGROUP       <1e> - <GROUP> B <ACTIVE>
```

## 2. Lecture des codes de service NetBIOS

Chaque entrée associe un nom à un **code de suffixe** (2 chiffres hex) qui indique le type de
service NetBIOS enregistré sous ce nom :

| Entrée | Code | Signification |
|--------|------|----------------|
| `METASPLOITABLE<00>` | Workstation Service | nom de la machine, service de base |
| `METASPLOITABLE<03>` | Messenger Service | reçoit les messages envoyés à ce nom |
| `METASPLOITABLE<20>` | **File Server Service** | **le partage de fichiers (SMB) est actif** |
| `WORKGROUP<00>` | nom du groupe de travail | `WORKGROUP` (confirmé) |
| `WORKGROUP<1d>` | Local Master Browser | la machine élue pour indexer le réseau local |
| `WORKGROUP<1e>` | Browser Election Service | participe aux élections de maître de groupe |

→ Sans avoir touché SMB, on sait déjà : le **nom de machine** (`METASPLOITABLE`), le
**domaine/groupe de travail** (`WORKGROUP`), et que le **service de partage de fichiers est
actif** (`<20>`) — confirmé ensuite en détail dans `samba-enum.md`.

## 3. Utilité pour la suite

Le nom de machine et le domaine récupérés ici servent de base pour construire une wordlist de
noms d'hôtes/domaines dans une attaque ultérieure (ex. formats d'authentification SMB du type
`DOMAINE\utilisateur`), même si dans ce labo l'énumération d'utilisateurs se fait plus
efficacement via RID cycling (`enum4linux`, voir `samba-enum.md`).

---

## Remédiation

- Désactiver NetBIOS (port 137/udp) si le partage de fichiers Windows/SMB legacy n'est pas
  nécessaire, ou le restreindre au réseau interne uniquement (pare-feu).
- Ne pas exposer de nom de machine ou de domaine interne sur un réseau non maîtrisé : cette
  information, bien que mineure isolément, facilite la reconnaissance (construction de
  wordlists, social engineering basé sur les conventions de nommage).

## Sources

- Session réelle (ce write-up) — `nmblookup -A`, recoupé avec `nmap --script nbstat`.

**Suite du parcours MS1 :** MySQL (port 3306), `root:root` ou vide selon config.
