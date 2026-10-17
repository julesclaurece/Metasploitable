# Write-up — Diagnostic port 8181 (Metasploitable 3)

Cible : `172.28.128.3:8181` · Pas de CVE — exercice de diagnostic réseau · Résultat : aucun
service, aucun processus, aucune trace sur le disque — conclusion différente du cas du port
3000, et tout aussi fondée sur preuve directe plutôt que supposition.

---

## 1. Constat initial

```bash
nmap -p 8181 172.28.128.3
```
```
8181/tcp closed intermapper
```
Même étiquette dans `ms3/parcours.md` que le port 3000 (« fermé de façon répétée — service
provisionné mais probablement crash-loopé »). Même rescan périodique appliqué :
```bash
for i in 1 2 3 4 5; do nmap -p 8181 172.28.128.3 | grep 8181; sleep 20; done
```
```
8181/tcp closed intermapper   (répété identique sur 5 cycles, ~100s)
```
Aucune alternance ici non plus.

## 2. Diagnostic via accès shell existant

Même méthode que pour le port 3000 — accès shell déjà obtenu via la backdoor UnrealIRCd
(`ms3/jules/unrealircd.md`), plutôt que de spéculer depuis l'extérieur :
```bash
ss -tlnp | grep 8181
```
```
(aucune ligne — rien n'écoute sur ce port, sur aucune interface)
```
```bash
ps aux | grep -iE "8181"
grep -rl "8181" /etc/init/ 2>/dev/null
```
```
(rien dans les deux cas)
```
Contrairement au port 3000 (service `sinatra` bien identifié dans `/etc/init/sinatra.conf`
et `/opt/sinatra/`), **aucune trace** d'un service destiné à ce port n'existe où que ce soit
sur le système : ni processus en cours, ni job `upstart` qui le mentionne, ni dossier
applicatif qui s'y réfère (`/opt/` ne contient que `apache_continuum`, `chatbot`, `chef`,
`docker`, `proftpd`, `readme_app`, `sinatra`, `unrealircd` — rien de lié à 8181).

## 3. Cause identifiée

**Ce service n'est pas provisionné dans cette instance de la VM**, point final — pas un
crash-loop (il n'y a rien à crasher), pas un problème de binding (il n'y a rien qui écoute
nulle part). La mention dans `ms3/parcours.md` reflète probablement soit un service
Metasploitable 3 documenté publiquement mais qui n'a pas été inclus dans le provisioning de
cette VM précise, soit une observation ponctuelle lors d'un scan antérieur (le port peut
avoir brièvement répondu pendant le tout premier démarrage de la VM, avant un crash
définitif suivi d'une absence totale de redémarrage — scénario compatible avec les artefacts
observés mais invérifiable après coup sans les logs de ce tout premier boot).

## 4. Portée de ce diagnostic

Deuxième occurrence du même exercice que le port 3000, avec une conclusion différente —
exactement l'objectif pédagogique annoncé par le parcours (« même exercice... pour ancrer le
réflexe »). Le réflexe confirmé ici : ne jamais conclure uniquement depuis l'extérieur
(`nmap` seul ne distingue pas « rien ici » de « quelque chose tourne mais mal exposé » — voir
le contre-exemple exact du port 3000) ; l'accès shell déjà obtenu permet de trancher avec
certitude dans les deux sens.

---

## Remédiation

Sans objet au sens strict (pas de service réellement présent à sécuriser) — le point à
retenir est méthodologique : documenter précisément quels services sont réellement
provisionnés dans une VM de labo évite ce genre d'ambiguïté lors d'audits ultérieurs.

---

**Fin du parcours MS3** — tous les services listés dans `ms3/parcours.md` ont été traités
(exploités avec succès, bloqués avec cause diagnostiquée, ou identifiés comme non
provisionnés).
