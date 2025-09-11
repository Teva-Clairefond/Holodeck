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

	    option domain-name "starfleet.lan";
	    option domain-name-servers 172.16.0.1;
    	subnet 172.16.0.1 netmask 255.255.0.0 {        # Déclare le réseau géré par le serveur DHCP 
    	range 172.16.0.100 172.16.0.200;        		# Plage IP pour les VM
        option routers 172.16.0.1;}	        	    # VM serveur comme passerelle

- `systemctl start isc-dhcp-server`

## IV - Installation du serveur DNS sur la VM serveur

-    `apt install bind9 bind9utils`
-    `nano /etc/bind/named.conf.options`

			acl "lan" {
		    172.16.0.0/16;
		    localhost;
		    localnets;
			};


			options {
		    // Répertoire de travail de Bind
		    directory "/var/cache/bind";

		    // Redirecteurs DNS (résolveurs externes)
		    forwarders {
		            1.1.1.1;
		            8.8.8.8;
		    };

		    // Mode récursif, pour résoudre les noms externes
		    recursion yes;

		    // Active la validation DNSSEC (vérifier l'authenticité des réponses DNS signées)
		    dnssec-validation auto;

		    // Ecouter sur toutes les interfaces réseau en IPv4 et IPv6
		    listen-on { any; };
		    listen-on-v6 { any; };

		    // Autoriser les requêtes pour les hôtes de l'ACL "lan"
		    allow-query { lan; };

			};

-  `named-checkconf`		# Vérifie la syntaxe du fichier dhcpd.conf
-  `nano /etc/bind/named.conf.local`	# Pour créer la nouvelle zone DNS
	
		zone "starfleet.lan" {
		        type master;
		        file "/etc/bind/db.starfleet.lan";
		        allow-update { none;};
		};


- `nano /etc/bind/db.starfleet.lan` #Création du fichier de zone directe

		$TTL    604800
		@       IN      SOA     VMserveur.starfleet.lan. root.starfleet.lan. (
		                              2         ; Serial
		                         604800         ; Refresh
		                          86400         ; Retry
		                        2419200         ; Expire
		                         604800 )       ; Negative Cache TTL
		;
		; Serveur DNS principal
		@           IN      NS      VMserveur.starfleet.lan.

		; Adresse du serveur DNS
		VMserveur   IN      A       172.16.0.1

		; Domaine racine pointe vers le serveur
		@           IN      A       172.16.0.1

		; Alias (facultatifs)
		dns         IN      CNAME   starfleet.lan.
		srv-dhcp    IN      A       172.16.0.1
  		www7            IN      A       172.16.0.1
		www8            IN      A       172.16.0.1
		php             IN      A       172.16.0.1



- `named-checkzone starfleet.lan /etc/bind/db.starfleet.lan` #Pour vérifier la syntaxe.

- `systemctl start bind9`  
- `sudo systemctl enable named.service`  
- `sudo systemctl status bind9`	#Pour vérifier que le serveur est bien actif


- `sudo nano /etc/resolv.conf`

		domain starfleet.lan
		nameserver 10.10.0.1
		





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

## VI - Installation du serveur web nginx

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
	Puis http://{nom-du-domaine}	#Vérifie l'attribution DNS


## VII - Passage en https

- `apt  install  openssl` 

-     Génération du certificat auto signé :
			mkdir  -p  /etc/nginx/ssl
	      openssl  req  -x509  -nodes  -days  365  -newkey  rsa:2048  \
	      -keyout  /etc/nginx/ssl/starfleet.key  \
	      -out  /etc/nginx/ssl/starfleet.crt

- Création du fichier de configuration HTTPS :
`nano  /etc/nginx/sites-available/starfleet-ssl.conf`

	    server  {
		listen  443  ssl;
		server_name  starfleet.lan;
		ssl_certificate  /etc/nginx/ssl/starfleet.crt;
		ssl_certificate_key  /etc/nginx/ssl/starfleet.key;
		root  /var/www/html;
		index  index.html;
		location  /  {
		try_files  $uri  $uri/  =404;
		}
		}
		server  {
		listen  80;
		server_name  starfleet.lan;
		return  301  https://$host$request_uri;
		}

-     Fait un lien entre le fichier de configuration du site et les sites activés : ln -s /etc/nginx/sites-available/starfleet-ssl.conf /etc/nginx/sites-enabled/
-   `Enlever le lien vers le le site de nginx par défaut : rm /etc/nginx/sites-enabled/default`
- `Vérifier la syntaxe et la validité des fichiers :
	nginx -t`

- `nano /etc/nginx/nginx.conf`
		

		user www-data;
		worker_processes auto;
		pid /run/nginx.pid;
	    include /etc/nginx/modules-enabled/*.conf;
		events {
		worker_connections 768;
		}

		http {
				sendfile on;
				tcp_nopush on;
				tcp_nodelay on;
				keepalive_timeout 65;
				types_hash_max_size 2048;
				include /etc/nginx/mime.types;
			    default_type application/octet-stream;
			    ssl_protocols TLSv1.2 TLSv1.3;
			    ssl_prefer_server_ciphers on;
			    access_log /var/log/nginx/access.log;
			    error_log /var/log/nginx/error.log;
			    server {
		        listen 443 ssl;
		        server_name starfleet.lan;

		        ssl_certificate     /etc/nginx/ssl/starfleet.crt;
		        ssl_certificate_key /etc/nginx/ssl/starfleet.key;

		        root /var/www/html;
		        index index.html;

		        location / {
	            try_files $uri $uri/ =404;
        }
	    }

	   server {
        listen 80;
        server_name starfleet.lan;
        return 301 https://$host$request_uri;
	    }
		}


- `systemctl restart nginx`


## VIII - Installation du serveur PHP


-     sudo wget -O /usr/share/keyrings/php-archive-keyring.gpg https://packages.sury.org/php/apt.gpg

- `nano /etc/apt/sources.list.d/php.list`

		deb [signed-by=/usr/share/keyrings/php-archive-keyring.gpg] https://packages.sury.org/php/ trixie main

-     apt install php7.4-fpm php8.2-fpm









