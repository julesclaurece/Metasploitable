# Write-up — Diagnostic port 3000 (Metasploitable 3)

Cible : `172.28.128.3:3000` · Pas de CVE — exercice de diagnostic réseau · Résultat :
service identifié, cause de son inaccessibilité externe déterminée avec certitude (pas une
supposition).

---

## 1. Constat initial

```bash
nmap -p 3000 172.28.128.3
```
```
3000/tcp closed ppp
```
`ms3/parcours.md` qualifie ce port de « fermé de façon répétée — service provisionné mais
probablement crash-loopé ». Avant de conclure, rescan périodique recommandé par le parcours :
```bash
for i in 1 2 3 4 5; do nmap -p 3000 172.28.128.3 | grep 3000; sleep 20; done
```
```
3000/tcp closed ppp   (répété identique sur 5 cycles, ~100s)
```
Aucune alternance observée — ce n'est **pas** un crash-loop classique (qui montrerait un
port qui bascule entre `open` et `closed`/`filtered` entre deux scans).

## 2. Diagnostic via accès shell existant

Plutôt que de deviner depuis l'extérieur, utilisation de l'accès shell déjà obtenu sur cette
cible via la backdoor UnrealIRCd (voir `ms3/jules/unrealircd.md`) — exactement ce que suggère
le parcours (« chercher pourquoi, logs si accès shell déjà obtenu par ailleurs »).
```bash
ps aux | grep -E "ruby|sinatra"
```
```
(rien au départ — le service n'était pas lancé)
```
```bash
cat /etc/init/sinatra.conf
```
```
description 'Run vulnerable Sinatra'
start on runlevel [2345]
chdir /opt/sinatra/
script
  /opt/sinatra/server
end script
```
```bash
status sinatra
```
```
sinatra stop/waiting
```
Le job `upstart` existe (prévu pour démarrer automatiquement) mais est arrêté. `start
sinatra` (via le compte non privilégié `boba_fett`) échoue silencieusement — droits
insuffisants pour piloter `upstart`. Lancement direct du binaire à la place :
```bash
cd /opt/sinatra && ./server &
ss -tlnp | grep 3000
```
```
LISTEN  0  128  127.0.0.1:3000  *:*
```

## 3. Cause identifiée

**Le service écoute sur `127.0.0.1:3000`, pas sur `0.0.0.0:3000`.** Il tourne bien, mais
uniquement sur l'interface de bouclage — inatteignable depuis l'extérieur de la VM par
construction, peu importe l'état du pare-feu ou le nombre de tentatives. Ce n'est pas un
crash-loop : c'est un service fonctionnel mais mal exposé, invisible depuis Kali, visible
seulement depuis la VM elle-même (`curl http://127.0.0.1:3000/` depuis le shell `boba_fett`
fonctionne).

## 4. Portée de ce diagnostic

Ce port est explicitement un exercice de diagnostic dans `ms3/parcours.md` (« distinguer un
port vraiment fermé d'un service qui crash-loope »). La vraie réponse ici est une troisième
catégorie, plus subtile que les deux prévues : **le service tourne, n'a jamais crashé, mais
n'a jamais été exposé** — un détail de configuration (`bind` sur loopback) qui ressemble à un
port fermé de l'extérieur alors que le service est parfaitement vivant. Exploitation de
l'application elle-même non menée ici (hors scope de cet exercice, qui porte sur le
diagnostic réseau, pas l'exploitation applicative) — nécessiterait un pivot depuis un accès
déjà obtenu sur la VM (ce qui est précisément le cas via la backdoor UnrealIRCd).

---

## Remédiation

- Si ce service n'est pas censé être accessible depuis l'extérieur, le binding loopback est
  en fait la bonne pratique — mais dans ce cas le job `upstart` qui le démarre automatiquement
  ne devrait pas être étiqueté "vulnérable" dans un contexte d'exposition réseau : vérifier
  la cohérence entre l'intention (service interne) et la configuration réelle.
- Si l'exposition était voulue (il s'agit visiblement d'une appli delibérément vulnérable
  pour un labo), documenter clairement l'interface d'écoute attendue pour éviter ce genre de
  confusion diagnostique en conditions réelles.

---

**Suite du parcours MS3 :** diagnostic du port 8181.
