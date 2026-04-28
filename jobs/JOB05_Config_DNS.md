# Configuration DNS

## Installation :

```bash
sudo apt install bind9 bind9utils
```

## Déclaration de la zone DNS :

```bash
sudo nano /etc/bind/named.conf.local
    zone "starfleet.lan" {      
        # Déclaration d'une zone DNS nommée starfleet.lan
        type master;    
            # Ce serveur est le maître (source) pour cette zone ; c’est lui qui contient le fichier de zone d’origine. (Autre valeur : slave pour esclave)
        file "/etc/bind/db.starfleet.lan"; 
            # chemin du fichier de zone qui contient les enregistrements DNS (A, NS, MX, etc.) pour cette zone.
    };
```


## Création du fichier de zone DNS

**IL FAUT ENLEVER LES COMMENTAIRES POUR QUE LE FICHIER FONCTIONNE !!!**

```bash
sudo nano /etc/bind/db.starfleet.lan
$TTL    604800
        # TimeTolive : Temps durant lequel le cache du client va conservé l'IP de sa machine cible, avant de redemander au resolveur DNS.
        # Un client demande à envoyer un paquet à pc1. Le serveur DNS local lui donne l'IP.
        # Puis le client va contacter pc1 avec son IP. Durant le TTL, le client n'a pas besoin de redemander l'ip au serveur DNS et 
        # peut directement envoyer au pc1.

@       IN      SOA     dns.starfleet.lan. admin.starfleet.lan. (
    # SOA : Start Of Authority indique le serveur qui a l'autorité
    # dns.starfleet.lan. : Serveur maître, serveur DNS du réseau local, nom de domaine FQDN (Fully Qualified Domain Name) qui commence à la racine (.)
    # admin.starfleet.lan : Stockage du mail de l'admin sous forme de nom de domaine, inutilisable sans outils extérieurs.
            2         ; Serial      
            # Num de version de la zone, dois être incrémenté à chaque modification
        604800         ; Refresh    
            # Si le numéro de série (serial) du maître est plus grand que celui que l’esclave connaît, 
            # ça veut dire que la zone a été modifiée.
            # L’esclave va alors télécharger la version mise à jour pour rester à jour
            # Ici, tous les 604800 secondes (7 jours), l'esclave check le serial du maître
        86400         ; Retry       
            # Si l'esclave (serveur secondaire) n'a pas pu contacter le maître (serveur DNS principal) 
            # alors il rééssayera dans 86400 secondes (1 jour).
        2419200         ; Expire    
            # Si l’esclave ne peut pas contacter le maître pendant la durée Expire, il considère sa copie obsolète
            # Dans ce cas : l’esclave arrête de répondre aux requêtes pour cette zone ou répond avec une erreur 
            # de serveur DNS (SERVFAIL)
        604800 )       ; Negative Cache TTL;    
            # Negative TTL : durée pendant laquelle une réponse négative (nom inexistant) peut être mise en cache.

@       IN      NS      dns.starfleet.lan.  ;
    # Définition du serveur DNS, par son nom de domaine FQDN

dns     IN      A       192.168.1.1    ;
    # Adresse IP locale du serveur DNS lui-même

vmcliente     IN      A       192.168.1.4
    # Hôte dans le LAN
    # Tous les noms que l'on souhaite résoudre doivent se trouver dans le fichier

ftp         IN      A       192.168.1.1
serveur     IN      A       192.168.1.1

chown root:bind /etc/bind/db.starfleet.lan
    # Le fichier a pour propriétaire l'utilisateur root et le groupe bind (groupe qui peut gérer le serveur DNS)
chmod 640 /etc/bind/db.starfleet.lan
    # Défini les permissions du fichier : rw- pour l'utilisateur ; r-- pour le groupe ; --- pour les autres
```


## Vérifier la configuration

```bash
named-checkconf
    # Vérifie la syntaxe des fichiers de configuration bind, si rien est affiché, pas d'erreur
named-checkzone starfleet.lan /etc/bind/db.starfleet.lan
    # named-checkzone <zonename> <fichier_de_zone> : vérifie la cohérence de la zone (format, SOA, enregistrements). 
    # Te retourne OK si tout va bien, sinon des erreurs indiquant la ligne fautive.
```

## Redémarrer le service et verifications :

```bash
sudo systemctl restart bind9

nslookup vmcliente.starfleet.lan 192.168.1.1
            # nslookup : outil de diagnostique DNS
            # vmcliente.starfleet.lan : Le nom de domaine que l'on souhaite résoudre
            # 192.168.1.1 : Adresse IP du serveur DNS a interroger
            # La commande envoie une requête DNS au serveur et affiche la réponse
```

## Faire en sorte que le DNS local redirige les noms inconnus (forwarders) :

```bash
nano /etc/bind/named.conf.options
    options {
            directory "/var/cache/bind";
                # Répertoire de travail de bind où sont stockées les données de cache et les fichiers temporaires
            recursion yes;
                # recursion yes; : permet au serveur de faire des requêtes récursives pour des noms qu’il ne connaît pas
            allow-query { any; };
                # Tous les clients peuvent formuler des requêtes DNS
            allow-recursion { localnets; localhost; };
                # allow-recursion { localnets; localhost; }; : ne permet la récursion qu’aux clients locaux (sécurité)

            forwarders {                # Partie intéressante : Elle fait la redirection vers les serveurs publiques pour CE serveur DNS,
                                        # mais de façon générale la redirection (par exemple pour une car NAT) se fait dans /etc/resolv.conf
                8.8.8.8;
                1.1.1.1;
            };
                # DNS publics vers lesquels BIND redirigera les requêtes inconnues
            
            dnssec-validation auto;
                # DNSSEC (Domain Name System Security Extensions) ajoute une signature cryptographique aux enregistrements DNS.
                # dnssec-validation auto; = “Active la vérification automatique de l’authenticité des réponses DNS signées par DNSSEC”.
                # Pourquoi ? : Si un client demande une résolution DNS au serveur (Nom -> IP) et qu'un attaqauant se place en tant que serveur DNS,
                # il peut rediriger vers une adresse ip frauduleuse et donc un sevreur frauduleux.

            listen-on { 127.0.0.1; 192.168.1.1; };
                # listen-on : définit les interfaces côté serveur qui peuvent écouter les requêtes
    };
```

## Captures d'écran :

DNS fonctionnel :
![DNS fonctionnel](/images/jobs/job05/DNS_fonctionnel.jpg)

Forwarding DNS des noms inconnus :
![Forwarding DNS des noms inconnus](/images/jobs/job05/forwarding_DNS.jpg)