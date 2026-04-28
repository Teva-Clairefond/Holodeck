# JOB03 - Configuration réseau VM serveur et DHCP


## Explications :

Nous allons maintenant installer le serveur DHCP sur la VM serveur.
Il configurera l'adresse ip de la VM serveur sur le réseau LAN.


## Installation et association du serveur DHCP à la carte réseau à laquelle on attribue une IP statique :

```bash
apt install isc-dhcp-server
    sudo nano /etc/default/isc-dhcp-server
        INTERFACESv4="ens34"
        INTERFACESv6=""

sudo nano /etc/network/interfaces
    auto ens34
        # auto : toujours au démarrage, même si l’interface est absente ou non branchée.
        # allow-hotplug : seulement quand l’interface est détectée (branchée / ajoutée). Voir fichier explicatif sur la différence entre auto et allow-hotplug
    iface ens34 inet static
    address 192.168.1.1       
        # Attribue une adresse IP locale statique à la carte LAN qui héberge le serveur DHCP
        # Ne dois pas être l'adresse du sous-réseau, donc ne dois pas terminer par .0 puisque c'est le sous-réseau qui termine de cette manière
    netmask 255.255.255.0
    gateway 192.168.1.2
        # Souvent dans un réseau NAT lié à des machines virtuelles, il y a la machine hôte sur X.X.X.1 et la gateway sur X.X.X.2. Ici nous sommes dans un réseau LAN et j'ai choisi de place l'adresse de nos serveurs sur 192.168.1.1
    dns-nameservers 192.168.1.1 
        # Il faut mettre l'ip de la machine qui servira de serveur DNS, ici en l'occurence c'est l'ip de la machine elle-même.
```


## Configuration du serveur DHCP :

```bash
sudo nano /etc/dhcp/dhcpd.conf

    option domain-name "starfleet.lan";
        # Nom du domaine (optionnel), correspond au nom complet des appareils du réseau (du domaine) exemple : imprimante.exemple.local  

    option domain-name-servers 192.168.1.1;
        # Adresse de la machine qui fait office de serveur DNS (ici la carte LAN locale)

    default-lease-time 600;
    max-lease-time 7200;
        # Durée du bail (lease) pour une IP

    subnet 192.168.1.0 netmask 255.255.255.0 {
        range 192.168.1.5 192.168.1.100;
            # La range ne doit pas couvrir l'adresse IP fixe du serveur DHCP pour qu'il n'y ait pas de conflits
        option routers 192.168.1.1;
        option broadcast-address 192.168.1.255;
    }
    # Définition du réseau à distribuer, c'est l'interface LAN qui représente le routeur côté réseau local
    
    host vmcliente {
        # "vmcliente" est juste un nom au sein du fichier DHCP ce n'est pas forcément le nom d'hôte de la machine sur le réseau. 
        # Puisque cela nécessite une résolution DNS
    hardware ethernet 00:0c:29:59:8f:c6;
        # Adresse MAC de la machine pour l'identification
    fixed-address 192.168.1.4;
        # Cette ligne sert à attribuer une adresse IP fixe à la machine
        # Je mets 192.168.1.4 car comme je vais configurer dans un premier temps l'adresse ip statique de la VM cliente sur 192.168.1.3 
    }

sudo systemctl restart isc-dhcp-server
```


## Captures d'écran :

![Configuration des interfaces réseau dans /etc/network/interfaces](/images/jobs/job03/config_interfaces_reseau.jpg)

![Interface du dhcp](/images/jobs/job03/interfaces_du_dhcp.jpg)

![Configuration du dhcp](/images/jobs/job03/Config_dhcp.jpg)