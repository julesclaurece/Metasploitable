# Captures — DVWA, injection de commande (MS2, port 80)

Cible : `http://192.168.56.101/dvwa/`. Captures **navigateur** (DVWA est une appli web).
2 captures.

## 01-login.png

Dans le navigateur : page de login DVWA, puis connexion réussie avec **`admin:password`**
(page d'accueil « Welcome to Damn Vulnerable Web Application », bouton Logout visible).

Encadrer en rouge : les identifiants `admin` / `password` saisis (ou le bandeau de bienvenue
après connexion).

## 02-command-injection.png

Page **Command Execution** (`/dvwa/vulnerabilities/exec/`), niveau de sécurité **low**.
Dans le champ de ping, saisir : `127.0.0.1;id` puis valider.

Encadrer en rouge : la ligne de sortie `uid=33(www-data) gid=33(www-data) ...` affichée sous
le formulaire (preuve de l'exécution de commande), et le bandeau `Security Level: low`.
