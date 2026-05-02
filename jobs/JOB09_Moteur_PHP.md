# Installation du moteur PHP :

Basé sur mon cours : https://github.com/Teva-Clairefond/Cours-generalites/blob/main/Cours/Linux%20generals/Installation%20d'un%20moteur%20PHP.md


## Explications :

Un serveur PHP n'est pas un un serveur à part entière, c'est en réalité un serveur web doté d'un moteur/interpréteur PHP.
Pour une page web static : index.html
Pour une page web avec du html et du php pour appeler des BdD ou donner de l'interactivité : index.php

1 - Le navigateur client envoie une requête au serveur web pour le fichier index.php
2 - le serveur web appelle le processus maître gestionnaire de processus, PHP-FPM, via une socket UNIX ou TCP 
3 - PHP-FPM réparti les requêtes FastCGI (il est fait pour appliquer FastCGI) vers les processus PHP-CGI (workers), qui contiennent chacun le moteur PHP
4 - Le moteur PHP interprête le fichier. Les parties html restent en html et les inclusions php sont transformées en html (ou autre).
5 - Chemin inverse vers le navigateur client

Le processus php-cgi est un executable binaire qui contient le moteur php

Le moteur php est le programme qui contient l’interpréteur PHP et toutes ses fonctionnalités.

Qu'est-ce qu'un interpréteur php ?
    Un fichier php est écrit en language php , illisible par la machine
    L'interpréteur php lit, ligne par ligne le fichier .php et donne un équivalent bianaire à la machine 

CGI (Common Gateway Interface) : Norme qui définit la communication entre le serveur web et les programmes externes (aussi appelés scripts CGI)
FastCGI : Version améliorée de CGI, plus rapide. Le processus php-cgi continue de tourner en fond au lieu d'être redémarré à chaque requête.

    
## Installation du site web php :

### 1) Installation des paquets :

**Différence par rapport à l'énoncé du sujet : Ma VM étant sous Debian13 Trixie, elle n'est pas compatible avec php7. Il n'y aura donc que php8.**

```bash
sudo apt update
curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb
echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/php.list
sudo apt install php8.4 php8.4-fpm php8.4-cli php8.4-mysql php8.4-curl php8.4-mbstring php8.4-xml php8.4-zip -y

systemctl status php8.4-fpm
```


### 2) Création du dossier pour le site web

```bash
sudo mkdir -p /var/www/www8
```


### 3) Création d'un fichier index.php pour le site 

```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/www8/index.php
```


### 4) Permissions

```bash
sudo chown -R www-data:www-data /var/www/www8
sudo chmod -R 755 /var/www
```


### 5) Génération du certificat auto-signé qui va servir pour les sites :

#### 5.1) Ajout de SAN (Subject Alternative Names) et fichier de configuration du certificat 
    
```bash
nano /tmp/nginx_openssl.cnf
    [ req ]
    distinguished_name = req_distinguished_name
    req_extensions = v3_req
    prompt = no

    [ req_distinguished_name ]
    C = FR
    ST = Île-de-France
    L = Paris
    O = MaSociete
    CN = www.starfleet.lan

    [ v3_req ]
    subjectAltName = @alt_names

    [ alt_names ]
    DNS.1 = www.starfleet.lan
    DNS.2 = www8.starfleet.lan
    IP.1 = 192.168.1.1
```



#### 5.2) Génération d'une clé privée + certificat auto-signé :

```bash
sudo openssl req -x509 -nodes -newkey rsa:4096 \
-keyout /etc/ssl/private/nginx.key \
-out /etc/ssl/certs/nginx.crt \
-days 3650 \
-config /tmp/nginx_openssl.cnf \
-extensions v3_req
```

#### 5.3) Vérifications

```bash
openssl x509 -in /etc/ssl/certs/nginx.crt -noout -text | grep -A2 "Subject Alternative Name"
```


### 6) Configuration dans sites-available


```bash
sudo nano /etc/nginx/sites-available/www8.starfleet.lan
    server {
        listen 80;
        listen [::]:80;

        server_name www8.starfleet.lan;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name www8.starfleet.lan;
        root /var/www/www8;
        index index.php index.html;

        ssl_certificate /etc/ssl/certs/nginx.crt;
        ssl_certificate_key /etc/ssl/private/nginx.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'HIGH:!aNULL:!MD5';
        ssl_prefer_server_ciphers on;

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        }

        access_log /var/log/nginx/starfleet.lan.access.log;
        error_log  /var/log/nginx/starfleet.lan.error.log;

        location / {
            try_files $uri $uri/ =404;
        }
    }
```

### 7) Activer le site

```bash
sudo ln -s /etc/nginx/sites-available/www8.starfleet.lan /etc/nginx/sites-enabled/
```
 
### 8) Ajout du site au resolveur DNS

Dans /etc/bind/db.starfleet.lan (Si serveur DNS local) ou dans /etc/hosts (Donne le bon ip sans passer par le DNS publique)
    
```bash
www8.starfleet.lan.   IN A 192.168.1.1
```

### 9) Vérification de la syntaxe

```bash
sudo named-checkzone starfleet.lan /etc/bind/db.starfleet.lan
```


### 10) Ajout du DNS local dans /etc/resolv.conf

```bash
nameserver 192.168.1.1               
    # ATTENTION : Le fichier resolv.conf définit l'ordre de test des serveurs DNS.
    # Si le premier serveur (1ere ligne) ne trouve pas ou passe au deuxième (2eme ligne)
```

### 11) Vérification de l'accès :


```bash
systemctl reload bind9    
curl -k https://www8.starfleet.lan
    # -k ignore le fait que le certificat soit auto-signé
```


## Installation de phpmyadmin :


phpmyadmin est un outil qui permet d'administrer une base de données au travers d'une interface graphique dans le navigateur. 

### 1) Installation du paquet :

```bash
apt install phpmyadmin -y
```

Il est alors demandé si possède un serveur web apache2 ou lighttpd : -> Ne rien sélectionner -> Entrer

Configure database for phpmyadmin with dbconfig-common? -> YES

### 2) Création du site Nginx pour phpmyadmin :

```bash
nano /etc/nginx/sites-available/php.starfleet.lan
    server {
        listen 80;
        listen [::]:80;

        server_name php.starfleet.lan;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name php.starfleet.lan;
        root /usr/share/phpmyadmin;
        index index.php index.html;

        ssl_certificate /etc/ssl/certs/nginx.crt;
        ssl_certificate_key /etc/ssl/private/nginx.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'HIGH:!aNULL:!MD5';
        ssl_prefer_server_ciphers on;

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        }

        access_log /var/log/nginx/starfleet.lan.access.log;
        error_log  /var/log/nginx/starfleet.lan.error.log;

        location / {
            try_files $uri $uri/ =404;
        }
    }
```

Créer le lien symbolique de sites-enabled vers sites-available :

```bash
ln -s //etc/nginx/sites-available/php.starfleet.lan /etc/nginx/sites-enabled/
```


### 3) Vérification : 


```bash
nginx -t
systemctl reload nginx
```

### 4) Ajout au DNS :

```bash
nano /etc/bind/db.starfleet.lan
    php  IN   A   192.168.1.1
```

Sans oublier d'incrémenter le Serial.

```bash
systemctl reload bind9
```

### 5) Création d'un utilisateur adminsql :

Afin de pouvoir nous connecter à la base de données depuis la machine cliente vers la machine serveur au travers de phpmyadmin, il nous faut un nouvel utilisateur admin. En effet, nous avons précédement dévalidé la connexion root distante.


```bash
mariadb
```

```sql
CREATE USER 'adminsql'@'localhost' IDENTIFIED BY '123'; 
    --mot de passe très faible pour facilité la manipulation, à ne pas reproduire
GRANT ALL PRIVILEGES ON *.* TO 'adminsql'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

### 6) Vérification :

Connexion depuis la machine cliente : https://php.starfleet.lan

utilisateur : adminsql
mot de passe : 123


## Administration de la VM serveur depuis admin.starfleet.lan avec Cockpit :

### Installation de Cockpit :


```bash
apt update
apt install cockpit -y
systemctl enable --now cockpit.socket

```


### 2) Création du site Nginx pour phpmyadmin :

```bash
nano /etc/nginx/sites-available/admin.starfleet.lan
    server {
        listen 80;
        listen [::]:80;

        server_name admin.starfleet.lan;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name admin.starfleet.lan;
        index index.php index.html;

        ssl_certificate /etc/ssl/certs/nginx.crt;
        ssl_certificate_key /etc/ssl/private/nginx.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'HIGH:!aNULL:!MD5';
        ssl_prefer_server_ciphers on;

    location / {
        proxy_pass https://127.0.0.1:9090;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_ssl_verify off;

        gzip off;
    }

        access_log /var/log/nginx/starfleet.lan.access.log;
        error_log  /var/log/nginx/starfleet.lan.error.log;
    }
```

Créer le lien symbolique de sites-enabled vers sites-available :

```bash
ln -s //etc/nginx/sites-available/admin.starfleet.lan /etc/nginx/sites-enabled/
```


### 3) Vérification : 


```bash
nginx -t
systemctl reload nginx
```

### 4) Ajout au DNS :

```bash
nano /etc/bind/db.starfleet.lan
    admin  IN   A   192.168.1.1
```

Sans oublier d'incrémenter le Serial.

```bash
systemctl reload bind9
```

## Captures d'écran :

Site php8 fonctionnel depuis la VM cliente : 

![Site php8 fonctionnel](/images/jobs/job09/site_php8_fonctionnel.jpg)

phpmyadmin fonctionnel depuis la VM cliente :

![Site phpmyadmin fonctionnel](/images/jobs/job09/phpmyadmin_fonctionnel.jpg)

Administration de la VM serveur possible via Cockpit sur admin.starfleet.lan  :

![Cockpit fonctionnel](/images/jobs/job09/admin_VM_serveur_cockpit.jpg)