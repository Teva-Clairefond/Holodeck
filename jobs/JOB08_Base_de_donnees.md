# Installation d'un serveur MariaDB (serveur SQL gestionnaire de base de données)

Je me base sur mon prope cours : https://github.com/Teva-Clairefond/Cours-generalites/blob/main/Cours/Linux%20generals/Installation%20d'un%20serveur%20MariaDB.md

## Explications : 

Le code inséré dans /var/www/index.php (par exemple) fait appel à une bibliothèque (comme mysqli par exemple).
Cette bibliothèque permet de communiquer avec le serveur MariaDB via le protocole MariaDB.

Les données de la base de données étant amenées à changer, elles sont situées dans /var/lib/mysql

Socket Unix : Fichier spécial qui permet à deux processus sur la même machine de communiquer, un peu comme un "port réseau interne" mais sans passer par TCP/IP.




## 1) Installation des paquets :

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-12.rolling"
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable mariadb
sudo systemctl status mariadb
```

## 2) Connexion en tant que root MariaDB et configuration:

```bash
sudo mysql
    # Ouvre l'invite de commande MariaDB
    # Connexion en tant que super utilisateur système au root MariaDB au travers de socket unix -> Pas besoin de mdp

sudo mariadb-secure-installation;
```
Script de sécurité pour modifier les options par défaut les moins sûres

1 - Identification en tant qu'utilisateur root de la BdD avec son mdp actuel : Cliquer sur "ENTER" (aucun mdp)
2 - Identification avec la socket UNIX, résultat : seul le compte root Linux peut devenir le root de la BdD.
3 - Création de mdp pour l'user root de la BdD ? : "N"
    # Pas besoin car root continuera de se connecter via le socket unix de la machine
4 - Pour toutes les questions suivantes il faut garder les valeurs par défaut : "Y"
    # Supprime les utilisateurs anonymes et la base de données de test, 
    # désactive les connexions root distantes, mais pas les non-root
    # chargera ces nouvelles règles afin que MariaDB implémente immédiatement les modifications que vous avez apportées.


### Les serveurs Web et MariaDB sont sur la même machine : 

### 3) Vérifications et tests :

```bash
sudo systemctl status mariadb

sudo mysqladmin version
    # Le but est de voir si l'on peut se connecter à la BdD. Ici on utilise l'outil mysqladmin qui est un client permettant d'executer des commandes d'admin.
    # Ici la commande consiste à se connecter en tant que root de la BdD et à renvoyer la version
```

## Capture d'écran :

![Mariadb fonctionnel](/images/jobs/job08/Mariadb_installed_and_working.jpg)