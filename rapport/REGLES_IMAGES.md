# Règles — Captures d'écran du rapport

Ce document couvre tout ce qui concerne les images du rapport : nommage des fichiers, et le
fichier `images.md` qui documente chaque capture. Pour la structure générale du rapport et
des pages HTML, voir [`REGLES.md`](REGLES.md). Pour la qualité des write-ups sources, voir
[`regle_writup.md`](../regle_writup.md).

---

## 1. Nommage des fichiers

Format : `NN-description-courte.png`.

**Pas de liste figée de noms à respecter** — le nombre de captures varie selon le service
(certains ont une étape de diagnostic en plus, d'autres vont droit à l'exploitation). La
règle est simple et s'applique à n'importe quel service :

- **`NN`** = position de la figure dans la page, dans l'ordre d'apparition (`01`, `02`,
  `03`...) — sert juste à trier les fichiers dans le dossier, lisible d'un coup d'œil.
- **`description-courte`** = 2-4 mots en minuscules, tirets, qui résument ce que montre
  *cette* capture précise (`recon-nmap`, `searchsploit`, `diagnostic-acl`,
  `exploit-msfconsole`, `session-root`...) — pas une case à cocher dans une liste prédéfinie.

**Numéro de fichier ≠ badge affiché dans le rapport.** Le nom de fichier reste numérique
(`01`, `02`...), mais le badge visible dans le `<figcaption>` (`<strong class="idx">`) est en
**lettres** (`a.`, `b.`, `c.`...) — pour ne pas se confondre avec la numérotation des
sections (`1. Identification`, `2. Méthode...`), qui va aussi de 1 à 6. "Voir capture b" ne
peut pas être pris pour "section 2", alors que "voir capture 2" le pouvait. Les deux
numérotations (fichier et badge) sont indépendantes — pas besoin de les faire correspondre.

Exemple réel (`vsftpd.html`, 6 captures) :
```
01-recon-nmap.png
02-recherche-module.png
03-info-cve.png
04-exploit-config.png
05-exploit-launch.png
06-session-root.png
```

Remarque sur cet exemple : chaque commande qui produit une information citée dans le texte a
sa trace visuelle — mais deux commandes très proches (`searchsploit` et `search vsftpd` dans
`msfconsole`, toutes deux de courtes recherches de confirmation) sont regroupées dans **une
seule** capture (`02-recherche-module`) plutôt que dupliquées en deux figures quasi
identiques. À l'inverse, l'exploitation (`exploit -z`) reste séparée en deux captures
distinctes (`04-exploit-config` pour la configuration des options, `05-exploit-launch` pour
le déclenchement et son résultat), parce que ce sont deux moments réellement différents de la
démarche. Règle de fond : regrouper ce qui se ressemble, séparer ce qui est distinct — pas de
figure par réflexe pour chaque commande.

Un service avec une étape de diagnostic en plus (ex. `distcc`) aurait simplement une capture
de plus, insérée à sa place chronologique, et toutes les suivantes renumérotées en
conséquence.

**Chaque capture doit montrer la commande tapée ET son résultat** — pas juste un bout de
sortie sans contexte. Capturer le terminal en entier (prompt visible), pas un recadrage trop
serré.

## 2. `images.md` — obligatoire dans chaque `images/<service>/`

Avant de prendre les captures (ou juste après), créer `images.md` dans le dossier
`images/<service>/` qui documente **chaque** image : son nom, la commande exacte qui produit
ce qu'elle montre, et — si nécessaire pour que le prof voie direct ce qu'il faut regarder —
quelle partie encadrer en rouge sur l'image avant de l'enregistrer.

Format, une section par capture :

```markdown
## 01-recon-nmap.png

Commande :
```bash
nmap -sV -p 21 172.28.128.5
```

Encadrer en rouge : la ligne de version (`vsftpd 2.3.4`)

## 02-searchsploit.png

Commande :
```bash
searchsploit vsftpd 2.3.4
```

Encadrer en rouge : aucun (résultat déjà court et lisible en entier)
```

- **Le write-up source peut rester en bloc** (ex. un resource script `cat > fichier.rc <<
  EOF ... EOF` lancé avec `msfconsole -q -r`) — c'est volontaire, ça permet de rejouer vite
  et de vérifier si la faille est toujours exploitable sans réfléchir à la présentation.
  **Les captures, elles, montrent la manipulation en interactif**, commande par commande
  (`use ...`, `set RHOSTS ...`, `set RPORT ...`, `exploit -z`...) comme le ferait un humain
  au clavier — plus clair à suivre pour le lecteur du rapport. Les deux formes doivent
  produire le **même résultat final**, mais pas être écrites à l'identique.
- "Encadrer en rouge" sert uniquement quand le résultat pertinent est noyé dans une sortie
  plus longue (ex. une ligne au milieu d'un scan nmap complet, un `uid=0` au milieu d'une
  sortie `id` plus verbeuse). Si toute la sortie capturée est déjà courte et pertinente,
  écrire "aucun" plutôt que d'encadrer par réflexe.
- `images.md` sert de checklist avant de commencer à prendre les captures : si une commande
  manque de contexte pour savoir quoi capturer exactement, c'est que le write-up source n'est
  pas assez précis — corriger le write-up d'abord (voir `regle_writup.md`).

## 3. Exemple de référence

Dossier complet à dupliquer comme modèle : [`images/vsftpd/`](images/vsftpd/) (6 captures +
son `images.md`).

## 4. Ce qu'on évite

- Pas de capture d'écran floue/illisible — retake plutôt que de mettre une image où le texte
  ne se lit pas.
