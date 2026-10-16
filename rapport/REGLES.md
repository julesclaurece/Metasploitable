# Règles de rédaction — Rapport de pentest Metasploitable (TP)

Ce document fixe les conventions pour que le rapport reste cohérent entre les deux
rédacteurs. À lire avant d'écrire une section.

Deux documents complémentaires : [`REGLES_IMAGES.md`](REGLES_IMAGES.md) (nommage des
captures d'écran, fichier `images.md`) et [`regle_writup.md`](../regle_writup.md) (qualité
des write-ups sources dans `ms1/`, `ms2/`, `ms3/`).

---

## 1. Organisation des fichiers

```
rapport/
├── REGLES.md              # ce fichier
├── index.html              # page d'accueil / sommaire du rapport
├── css/
│   └── style.css           # une seule feuille de style partagée
├── services/
│   ├── vsftpd.html          # une page HTML par faille exploitée
│   ├── distcc.html
│   ├── ftp-modcopy.html
│   └── ...
└── images/
    ├── vsftpd/
    │   ├── images.md            # consignes de capture — voir REGLES_IMAGES.md
    │   ├── 01-recon-nmap.png
    │   ├── 02-recherche-module.png
    │   ├── 03-info-cve.png
    │   ├── 04-exploit-config.png
    │   ├── 05-exploit-launch.png
    │   └── 06-session-root.png
    ├── distcc/
    └── ...
```

- **Un dossier d'images par service**, nommé comme le fichier HTML correspondant.
- **Une page HTML par faille exploitée** — pas un seul fichier géant.
- `index.html` liste tous les services traités avec un lien vers chaque page.

## 2. Qui écrit quoi

Avant de commencer une section, annoncer à l'autre quel service tu prends (éviter que deux
personnes écrivent la même page en même temps → conflit).

Convention simple : chacun travaille sur ses propres fichiers (`services/<nom>.html` +
`images/<nom>/`) — pas d'édition croisée du fichier de l'autre sans prévenir.

## 3. Structure obligatoire d'une page de service (`services/<nom>.html`)

Chaque page doit contenir, dans cet ordre, ces 6 sections (calquées sur
[`services/vsftpd.html`](services/vsftpd.html), à suivre à la lettre) :

1. **Identification** — service, port, version, cible (IP) ; inclut la capture du scan de
   reconnaissance (`nmap`) avec explication de ce qui a été cherché
2. **Méthode d'identification de la CVE et du module** — pas juste "contexte" : montrer la
   vraie recherche (`searchsploit`, puis `search <produit>` dans `msfconsole` pour le chemin
   du module, puis `info <module>` pour la CVE), **chaque commande avec sa vraie sortie —
   via la capture d'écran qui l'accompagne, pas recopiée en `<pre>` dans le texte** (voir
   §4 "Ton et style" sur ce point précis), puis l'explication du mécanisme de la faille (voir
   `regle_writup.md` §1 — aucune info ne sort de nulle part)
3. **Démarche d'exploitation** — captures + commandes utilisées, avec explication de ce que
   fait chaque commande (le nom de la commande en `<code>` inline dans le texte suffit, la
   capture juste en dessous montre son exécution complète)
4. **Preuve d'accès obtenu** — capture claire du résultat (ex. `id` montrant `uid=0`)
5. **Remédiation** — **obligatoire**, souvent oublié : comment corriger cette faille en
   situation réelle (mise à jour, changement de config, désactivation du service...)
6. **Sources** — write-up technique correspondant dans `ms1/<jules|josa>/`,
   `ms2/<jules|josa>/` ou `ms3/<jules|josa>/` du dépôt (lien relatif, ex.
   `../../ms2/jules/vsftpd.md`), + toute CVE/référence externe citée

## 4. Ton et style d'écriture

- Rédaction en **français**, phrases complètes (pas de style télégraphique façon notes
  perso).
- **Expliquer, pas juste montrer.** Une capture d'écran sans explication de ce qu'elle
  prouve n'a pas sa place dans le rapport.
- **Ne jamais dupliquer une capture en texte.** Si une commande et son résultat sont déjà
  montrés par une `<figure class="capture">`, ne pas les reproduire dans un `<pre
  class="terminal">` juste avant — le texte autour **renvoie** à la capture ("capture 2
  ci-dessous", "voir capture 4") plutôt que de la recopier. `<pre class="terminal">` ne sert
  que pour un extrait **sans** capture correspondante (rare).
- Pas de jargon non défini à la première occurrence (ex. la première fois que "RCE"
  apparaît dans une page, l'écrire en toutes lettres : "exécution de code à distance
  (RCE)").
- CVE toujours citées sous la forme `CVE-XXXX-XXXXX` avec un lien vers
  `https://nvd.nist.gov/vuln/detail/CVE-XXXX-XXXXX`.

## 5. Ce qu'on NE met PAS dans le rapport

- Pas de vrais mots de passe/secrets autres que les identifiants par défaut documentés des
  VM volontairement vulnérables (`msfadmin:msfadmin`, etc. — publics par nature).
- Pas de contenu copié tel quel depuis un write-up externe trouvé en ligne — reformuler avec
  ses propres mots même si la source a servi à comprendre.

(Pour les captures d'écran, voir [`REGLES_IMAGES.md`](REGLES_IMAGES.md).)

---

## 6. Template de référence

Le style est déjà fait (`css/style.css`, partagé) — **ne pas en créer un autre**. Une page
d'exemple complète existe : [`services/vsftpd.html`](services/vsftpd.html). Pour une
nouvelle page, dupliquer ce fichier, remplacer le contenu section par section, garder la
structure (classes CSS, ordre des sections) identique.

Checklist rapide pour une nouvelle page :
1. Copier `services/vsftpd.html` → `services/<nom-du-service>.html`
2. Créer `images/<nom-du-service>/` + son `images.md` (voir
   [`REGLES_IMAGES.md`](REGLES_IMAGES.md) — lister les captures prévues et leurs commandes
   **avant** de les prendre, pas après)
3. Prendre les captures en suivant `images.md`
4. Remplir les 6 sections (§3) avec le contenu réel
5. Ajouter une entrée dans la liste de `index.html` (`<ul class="sommaire-list">`)
6. Vérifier l'affichage dans un navigateur avant de commit

---

*Prochaine étape : chacun prend un service (voir §2), duplique le template, et commence à
remplir.*
