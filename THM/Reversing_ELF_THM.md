🛠️ TryHackMe - Reversing ELF Writeup

Bienvenue dans ce writeup de la room Reversing ELF sur TryHackMe

## Crackme1

./crackme1
flag{REDACTED}

Binaire très simple, flag affiché directement

## Crackme2

Analyse avec strings :

strings ./crackme2

Indices repérés :

Usage: %s password
super_secret_password
Access denied.
Access granted.

Le mot `super_secret_password` est comparé en clair via strcmp

Tester le mot de passe :

./crackme2 super_secret_password

Résultat :

Access granted.
flag{REDACTED}

Leçon : certains mots de passe sont codés en dur, pas besoin de déboguer

## Crackme3

Analyse avec strings :

strings ./crackme3

On y trouve une chaîne base64 :

ZjByX3kwdXJfNWVjMG5kX2xlNTVvbl91bmJhc2U2NF80bGxfN2gzXzdoMW5nNQ==

Décodage :

echo "ZjByX3kwdXJfNWVjMG5kX2xlNTVvbl91bmJhc2U2NF80bGxfN2gzXzdoMW5nNQ==" | base64 -d

Résultat :

REDACTED

Tester le mot de passe :

./crackme3 REDACTED

Correct password, flag affiché

Leçon : malloc sert à l’allocation mémoire, les messages `"malloc failed"` sont là pour la sécurité

## Crackme4

Analyse avec gdb :

gdb ./crackme4
init-pwndbg
start

Le programme attend un argument ligne de commande :

Usage : ./crackme4 password
This time the string is hidden and we used strcmp

Lister les fonctions :

info functions

Fonctions intéressantes : compare_pwd, get_pwd, main et strcmp\@plt

Breakpoint sur strcmp et exécution :

b \*0x400520
run test

Lire registres RDI et RSI :

0x7fffffffdc00: "my_m0r3_secur3_pwd"
0x7fffffffe132: "test"

Tester le mot de passe :

./crackme4 my_m0r3_secur3_pwd
password OK

Leçon : on peut détourner strcmp pour récupérer le mot de passe codé en dur

## Crackme5

Analyse avec strings et ltrace :

strings ./crackme5
ltrace ./crackme5

Chaînes observées : Enter your input:, Good game, Always dig deeper, strncmp

Comparaison directe révèle le mot de passe :

REDACTED

Tester :

./crackme5

Résultat : Good game

Leçon : strncmp compare les chaînes, ltrace permet de suivre la logique

## Crackme6, Crackme7 et Crackme8

Techniques supplémentaires : conversions hex ↔ déc, menus, arguments négatifs

Exemple Crackme7 :

./crackme7  
Menu:
[1] Say hello
[2] Add numbers
[3] Quit

[>] 31337
Wow such h4x0r!
flag{REDACTED}

Exemple Crackme8 :

./crackme8 -889262067
Access granted.
flag{REDACTED}

🧠 Leçons principales

- strings pour extraire les chaînes cachées
- Base64 pour mots de passe encodés
- strcmp et strncmp pour comparer l’entrée à des valeurs codées
- malloc et messages `"malloc failed"` pour la sécurité mémoire
- gdb et ltrace pour l’analyse dynamique
- conversions de valeurs et utilisation de menus ou arguments spécifiques

Cette room est un excellent exercice pour apprendre le reverse engineering et le cracking de binaires ELF simples
