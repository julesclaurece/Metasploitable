# Write-up — rpcbind : énumération (Metasploitable 3, port 111)

Cible : `172.28.128.3:111` · Pas de CVE — exercice d'énumération RPC · Résultat : service
confirmé actif et correctement lié à toutes les interfaces, mais inaccessible depuis
l'extérieur — cause isolée au niveau réseau, pas au niveau du service lui-même.

Tout ce qui suit est copier-coller direct (testé en zsh), côté 🖥 Kali uniquement.

---

## 1. Recon

```bash
nmap -sC -sV -p 111 172.28.128.3
```
```
111/tcp filtered rpcbind
```
`ms3/parcours.md` note ce port comme « vu de façon intermittente ». `filtered` (et non
`closed`) signifie que les paquets sont silencieusement ignorés — pas de RST reçu —
contrairement à un port réellement fermé qui répondrait activement.

```bash
rpcinfo -p 172.28.128.3
```
Aucune sortie, la commande ne se termine jamais (tuée manuellement après plusieurs minutes).

```bash
nmap -sU -p 111 172.28.128.3
```
```
111/udp open|filtered rpcbind
```
Même ambiguïté côté UDP — `nmap` lui-même ne peut pas trancher entre « ouvert » et
« filtré » depuis l'extérieur, faute de réponse exploitable dans les deux sens.

## 2. Diagnostic via accès shell existant

Plutôt que de deviner, vérification directe via l'accès shell déjà obtenu sur la cible
(backdoor UnrealIRCd, voir `ms3/jules/unrealircd.md`) :
```bash
ss -lunp | grep 111    # UDP
ss -ltnp | grep 111    # TCP
ps aux | grep -i rpc
```
```
UNCONN  *:111  (UDP)
LISTEN  *:111  (TCP)
root  919  rpcbind
root  497  rpc.idmapd
statd 943  rpc.statd -L
```
**`rpcbind` tourne bel et bien, en root, et écoute correctement sur `*:111`** — toutes les
interfaces, TCP et UDP. Le service n'est ni crashé ni mal lié (contrairement au cas du port
3000, voir `ms3/jules/port-3000-diagnostic.md`).

Tentative de lire les règles de pare-feu locales pour confirmer un blocage réseau explicite :
```bash
iptables -L -n
```
```
can't initialize iptables table 'filter': Permission denied (you must be root)
```
Le compte `boba_fett` (obtenu via la backdoor UnrealIRCd) n'est pas root — impossible de
lire la règle exacte qui bloque ce port depuis cette session. Mais la combinaison des
observations (service actif + correctement lié + aucune réponse externe sur TCP et UDP)
élimine les deux autres hypothèses possibles (service absent, service mal lié) et ne laisse
que le filtrage réseau comme explication cohérente.

## 3. Conclusion

**rpcbind est vivant et correctement configuré** — le blocage est réseau, pas applicatif,
confirmé par élimination méthodique plutôt que supposé. Ce comportement intermittent
(signalé dans `parcours.md`) est cohérent avec une règle de filtrage qui laisse parfois
passer un paquet isolé (d'où les observations ponctuelles passées) sans jamais permettre un
échange complet (d'où l'impossibilité de faire une vraie énumération RPC avec
`rpcinfo -p`).

---

## Remédiation

Sans objet côté application — `rpcbind` est correctement configuré ici. Si le filtrage
réseau observé est intentionnel (bonne pratique : rpcbind n'a aucune raison d'être exposé en
dehors du réseau de confiance d'un serveur NFS/NIS), le documenter clairement ; sinon,
vérifier la cohérence de la règle de pare-feu avec l'intention réelle du service.

---

**Fin du parcours MS3** — tous les services listés dans `ms3/parcours.md` ont été traités :
exploités avec succès, bloqués avec cause diagnostiquée, ou identifiés comme non
provisionnés/filtrés au niveau réseau.
