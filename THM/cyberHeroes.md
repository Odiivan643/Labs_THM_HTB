Présentation du lab

Nom : CyberHeroes
Catégorie : Web Security
Objectif : Identifier les services exposés, analyser l'application web et comprendre son mécanisme d'authentification.
Outils utilisés : Nmap, navigateur web, outils de développement.
Statut : Lab Résolu

1. Reconnaissance — Scan Nmap

Objectif
La première étape consiste à identifier les ports ouverts, les services actifs et les versions des logiciels exposés par la machine cible.

Commande utilisée
nmap -sC -sV -oN scancyberheroes.txt adresse_ip_cible

Explication des options

-sC : Exécute les scripts Nmap par défaut.
-sV : Détecte les versions des services.
-oN scancyberheroes.txt : Enregistre le résultat au format texte normal dans ce fichier.

Résultat du scan

<img width="685" height="428" alt="image" src="https://github.com/user-attachments/assets/7fa3d74b-7384-422e-932b-a42039e063a1" />

Analyse

À partir des résultats, je vais identifier les ports ouverts, les services accessibles et les éventuels services web à examiner plus en détail.

2.  Analyse de l'application web

Après le scan, je me suis rendu sur la page web hébergée par la machine cible.

Observations

URL : adresse_ip_cible : 80 [ ip:port ]
Page découverte : /login 

2.1. Récupération du contenu de la page

J'ai utilisé curl pour récupérer les en-têtes HTTP ainsi que le contenu retourné par la page de connexion.

Commande utilisée 
curl -i http://IP_CIBLE/login.html

Explication :

curl : permet d'effectuer des requêtes HTTP depuis le terminal.
-i : affiche les en-têtes HTTP en plus du corps de la réponse.
http://IP_CIBLE/login.html : correspond à la ressource web ciblée

L'inspection du contenu HTML m'a permis de découvrir une fonction JavaScript nommée authenticate()

<img width="781" height="522" alt="image" src="https://github.com/user-attachments/assets/19119e9f-0b7e-4a3b-b56f-bc87fa8d415a" />

2.2. Analyse du code JavaScript

La fonction authenticate() récupère les valeurs saisies dans les champs du formulaire :

a = document.getElementById('uname')
b = document.getElementById('pass')

Ces instructions correspondent respectivement au nom d'utilisateur et au mot de passe.
J'ai ensuite identifié une fonction qui inverse une chaîne de caractères :

const RevereString = str => [...str].reverse().join('');

Cette fonction transforme une chaîne en inversant l'ordre de ses caractères. Il ne s'agit donc pas d'un chiffrement, mais d'une simple transformation réversible.

2.3. Identification des identifiants

La condition suivante compare les valeurs saisies avec des valeurs présentes directement dans le JavaScript :

if (
    a.value == "h3ck3rBoi" &
    b.value == RevereString("54321@terceSrepuS")
) {

Voici ce que font ses différentes parties :
- [...str] : transforme la chaîne en tableau de caractères.
- .reverse() : inverse l'ordre des caractères.
- .join('') : rassemble les caractères pour reformer une chaîne.
- 
L'analyse m'a permis d'identifier le nom d'utilisateur et de retrouver le mot de passe en inversant la chaîne fournie.

python3 -c 'print("54321@terceSrepuS"[::-1])'

Identifiants obtenus :
SuperSecret@12345

Cette découverte montre que les identifiants étaient exposés dans le code côté client et pouvaient être retrouvés sans effectuer d'attaque par dictionnaire.

2.4. Analyse de la récupération du flag

Après validation des identifiants, la fonction JavaScript envoie une requête HTTP GET pour récupérer un fichier dont le nom est construit dynamiquement.

2.4. Récupération du flag

Congrats Hacker, you made it !! Go ahead and nail other challenges as well :D flag{edb0be532c540b..................}

<img width="695" height="547" alt="image" src="https://github.com/user-attachments/assets/60b8dfb4-3519-4777-8f78-1a9620e45b20" />

