# Holodeck



## I - Création des cartes réseaux dans VirtualBox 

 - VM serveur :
        Ajout d'une carte réseau WAN en utilisant NAT
        Ajout d'une carte réseau LAN en utilisant un réseau local 
 - VM Client :
           Ajout d'une carte réseau LAN en utilisant un réseau local
 - Aller dans l'onglet `réseau` > `adapter` > `Configurer la carte
   manuellement` : Rentrer une ip de catégorie B
 - Aller dans l'onglet `réseau` > `Serveur DHCP` > Décocher `activer le
   serveur`


## II - "Pas de compte sudo, conformément aux directives de Starfleet."

 - `su -`    #Devient root
               `visudo /etc/sudoers`  #Accède au fichier sudoers
 - Dans le fichier sudoers
                   `teva ALL=(ALL) /usr/bin/apt`    #Eleve les droits de l'utilisateur spécifiquement pour apt install, sans faire partie du groupe sudo
 - `su - teva`   #Redevient l'utilisateur
 - Dans le fichier .bashrc :
                   `alias apt='sudo apt'`    #Fait en sorte que le shell interprète 'sudo apt' quand on marque juste 'apt'
 - `source ~/.bashrc`    #Applique les changements de bashrc

## III - Mise en place d'un serveur DHCP sur la VM serveur :
    

 - `apt install isc-dhcp-server`     #Installation du serveur DHCP
 
 - `nano /etc/default/isc-dhcp-server`
           `INTERFACESv4="enp0s8"`       #Relie le serveur DHCP au réseau LAN

 - `nano /etc/dhcp/dhcpd.conf`
        `subnet 172.16.0.1 netmask 255.255.0.0 {`            #Déclare le réseau géré par le serveur DHCP 
        `range 172.16.0.100 172.16.0.200;`        # Plage IP pour les VM
        `option routers 172.16.0.1;}`         # VM serveur comme passerelle

- `systemctl start isc-dhcp-server`

## IV - Installation du serveur DNS sur la VM serveur

 - `apt install bind9`

 - `nano /etc/bind/named.conf.local`
 

	   zone "startfleet.lan" {
       type master;
       file "/etc/bind/db.starfleet.lan";
       allow-update { none;};
       };
        
       zone ".in-addr.arpa" {
       type master;
       file "/etc/bind/db.inverse.starfleet.lan";
       allow-update { none;};
       };
    
- `nano db.starfleet.lan`

## V - Connecter les deux VM via le LAN

- Dans la VM serveur :

	-	  nano /etc/network/interfaces
	      auto enp0s8
          iface enp0s8 inet static
          address 172.16.0.1       #Attribue une adresse IP locale statique à la VM serveur
          netmask 255.255.0.0

	- `systemctl restart networking`

- Dans la VM client :
	   
	- 	  nano /etc/network/interfaces
	      auto enp0s3                 #enp0s3 car il n'y a qu'une seule carte réseau sur la VM cliente
	      iface enp0s3 inet dhcp      #L'addresse IP locale sera attribuée par le serveur DHCP à la VM cliente

	- `systemctl restart networking`

	-      ip addr     #Verifier l'attribution d'une nouvelle adresse ip

## VI - Installation du serveur web nginx :

   - `sudo apt install curl gpg`
   -  `curl -fsSL https://nginx.org/keys/nginx_signing.key \ | sudo gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg`
   - `sudo nano /etc/apt/sources.list.d/nginx.list`
	
		    deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian trixie nginx
       
  - `apt install nginx`
  - `apt policy nginx (pour vérifier que c'est le bon nginx qui est installé)`
  - `apt install iptables`
  - `iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT`
  - `systemctl start nginx`
  - `systemctl status nginx`
    
    Dans la machine cliente, sur le navigateur : http://{adresse ip de la machine serveur sur le LAN}   #Vérifier la connexion des VM et le serveur nginx




