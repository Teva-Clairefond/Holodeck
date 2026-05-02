# Holodeck

Projet de mise en place d'une infrastructure web complète sur deux machines
virtuelles Debian.

L'objectif est de fournir un environnement serveur sécurisé pour héberger des sites web, gérer les utilisateurs et tester les services depuis une VM cliente.

## Architecture

- **VM serveur** : Debian sans interface graphique, 2 Go RAM, 2 vCPU, disque
  32 Go, deux cartes réseau WAN/LAN.
- **VM cliente** : Debian avec interface graphique, 2 Go RAM, 2 vCPU, disque
  16 Go, connectée au LAN du serveur avec un navigateur web.

## Services à mettre en place

- DHCP et DNS sur le domaine `starfleet.lan`
- Nginx en HTTPS
- PHP 7.x et PHP 8.x en parallèle
- MariaDB et phpMyAdmin
- FTP sécurisé en SSL/TLS, chrooté sur le dossier web
- Annuaire LDAP pour l'authentification
- Interface d'administration de la VM
- Pare-feu autorisant uniquement les ports nécessaires

## Noms DNS attendus

- `www8.starfleet.lan` : site web en PHP 8
- `php.starfleet.lan` : phpMyAdmin
- `admin.starfleet.lan` : administration de la VM
- `vscore.starfleet.lan` : Visual Studio Code Server, option avancée

## Contraintes

- Aucun compte sudo.
- Nginx, PHP et MariaDB doivent être installés dans une version récente, hors
  dépôts Debian par défaut.
- Le serveur web et le serveur FTP doivent utiliser un certificat SSL.
- Les services doivent être documentés et testés depuis la VM cliente.

## Documentation attendue

Le dépôt contient les procédures d'installation et de configuration pour :

- les deux machines virtuelles Debian ;
- le réseau WAN/LAN, DHCP et DNS ;
- Nginx, PHP, MariaDB et phpMyAdmin ;
- FTP sécurisé ;
- LDAP ;
- l'administration distante ;
- VS Code Server ;
- l'exportation des VMs ;
- la sauvegarde de la configuration complète du serveur.

## Compétences visées

- Administrer et sécuriser des infrastructures systèmes.
- Concevoir une solution technique répondant à un besoin d'évolution
  d'infrastructure.
- Participer à l'élaboration et à la mise en oeuvre d'une politique de
  sécurité.
