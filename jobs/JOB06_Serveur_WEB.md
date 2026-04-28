# JOB06 - Installation d'un serveur WEB NGINX

Basé sur mon cours : https://github.com/Teva-Clairefond/Cours-generalites/blob/main/Cours/Linux%20generals/Installation%20d'un%20serveur%20web%20Nginx.md

## Explications :

Les fichiers de configuration des sites sont dans : /etc/nginx/sites-available/
Pour activer un site on crée un lien symbolique dans : /etc/nginx/sites-enabled/
Le répertoire racine par défaut est : /var/www/html (mais on créera /var/www/starfleet.lan). C'est ici que se trouve le contenu du site.
Il est placé dans /var car les infos du sites sont souvent amenées à changer.


## Serveur web HTTP :

### Installation de Nginx :

```bash
sudo apt update
sudo apt upgrade

sudo apt install apt-transport-https curl gnupg2 lsb-release
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null    

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/mainline/debian $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list

sudo apt update
sudo apt install -y nginx
```



### Création du répertoire racine web dans /var/www/starfleet.lan/html et du fichier de configutation du site dans /etc/nginx/sites-available/ :

```bash
sudo mkdir -p /var/www/starfleet.lan/html
    # -p : mkdir crée toute la hiérarchie de dossiers nécessaires et n’affiche pas d’erreur si le dossier existe déjà.

sudo chown -R $USER:$USER /var/www/starfleet.lan/html
    # Donne la propriété de tous les éléments du dossier html à l'utilisateur

sudo chmod -R 755 /var/www/starfleet.lan
    # droits lecture/exécution pour tous et écriture pour le propriétaire pour les éléments du dossier starfleet.lan

echo '<!doctype html><title>starfleet.lan</title><h1>It works!</h1>' > /var/www/starfleet.lan/html/index.html
    # Créer une page de test
    # > (redirection de sortie) prend cette sortie et l’écrit dans un fichier (/var/www/starfleet.lan/html/index.html).
        # si le fichier n’existe pas → il est créé automatiquement.
        # si le fichier existe déjà → il est écrasé (contenu remplacé).
    # index.html (ou index.php si le site utilise php) est la page par défaut du site

nano /etc/nginx/sites-available/starfleet.lan
    server {
        # Un bloc serveur correspond à un site sur le serveur. Si il y a plusieurs sites, il y a plusieurs blocs server {}

        listen 80;
            # Le port 80 correspond aux connexions HTTP
        listen [::]:80;                  
            # [::] veut dire toutes les adresses IPv6 disponibles
            # listen [::]:80 = accepte les connexions IPv6 sur le port 80.
        
        server_name starfleet.lan www.starfleet.lan;
            # La résolution du nom de domaine en l'IP du serveur se fait par le DNS du client. Mais un serveur peut avoir plusieurs sites.
            # C'est là qu'intervient serveur name, qui permet de rediriger vers le bon site en fonction de la requête du client

        root /var/www/starfleet.lan/html;  
            # Chemin vers la racine des fichiers web (contenu du site)
        index index.html index.htm;      
            # fichiers index possibles : nginx cherche d’abord index.html
                # s’il n’existe pas → il teste index.htm
                # s’il n’existe pas → il teste index.php


        access_log /var/log/nginx/starfleet.lan.access.log;  
        error_log  /var/log/nginx/starfleet.lan.error.log;   
            # Chemin des fichiers où seront stockés les logs d'accès et d'erreurs

        location / {
            try_files $uri $uri/ =404;   
                # règle de sécurité : renvoie 404 si fichier introuvable
        }
    }
```


VERSION COPIABLE :

server {
    listen 80;
    listen [::]:80;                  
    server_name starfleet.lan www.starfleet.lan;
    root /var/www/starfleet.lan/html;  
    index index.html index.htm;      

    access_log /var/log/nginx/starfleet.lan.access.log;  
    error_log  /var/log/nginx/starfleet.lan.error.log;   

    location / {
        try_files $uri $uri/ =404;   
    }
}



### Activation et vérifications :

```bash
sudo ln -s /etc/nginx/sites-available/starfleet.lan /etc/nginx/sites-enabled/
    # ln -s : créer un lien symbolique (symlink)
        # C'est un raccourci qui pointe vers le fichier original dans sites-available
    # ln : créer un lien physique
        # le fichier1 et le fichier2 pointent vers le même contenu sur le disque. Ce sont des entrées différentes vers ce contenu.
        # Si l'un est supprimé cela n'affecte pas l'autre fichier. 
        # Par contre si le contenu est modifié, il est modifié quel que soit le fichier (quel que soit l'entrée vers le contenu)

sudo nginx -t
    # teste la syntaxe de tous les fichiers de config nginx

sudo systemctl reload nginx
curl starfleet.lan
```



## Passage en HTTPS avec un certificat auto-signé :


### Ajout de SAN (Subject Alternative Names) et fichier de configuration du certificat

```bash
nano /tmp/nginx_openssl.cnf
    # C'est un fichier temporaire qui ne sert qu'une fois : Au moment de générer le certificat
    # Une fois le certificat (.crt) et la clé privée (.key) générés, il n’y a plus besoin du fichier de config
    # Si on veut générer plusieurs certificats identiques, il faut le mettre dans /etc/ssl par exemple

    [ req ]
        # Section principale qui dit à OpenSSL comment générer la requête de certificat (CSR) ou un certificat auto-signé
        # Si on fait appel à une certification extérieur, dans ce cas on génèrera une CSR, dans l'autre cas ça sera un certif auto-signé
    distinguished_name = req_distinguished_name
        #distinguished_name = req_distinguished_name → indique la section où sont définis les champs d’identité du certificat (DN)
    req_extensions = v3_req
        # req_extensions = v3_req → précise quelle section contient les extensions X.509 à ajouter (dans la section on ajoutera le SAN)
    prompt = no
        # OpenSSL n’affiche pas le questionnaire sur les infos du DN et prend directement les valeurs écrites dans la section [ req_distinguished_name ]

    [ req_distinguished_name ]
    C = FR
    ST = Île-de-France
    L = Paris
    O = MaSociete
    CN = ftp.starfleet.lan
        # CN = Common Name (doit correspondre au nom DNS que les clients utiliseront, ex. ftp.starfleet.lan)
        # L'ensemble de ces infos (Pays, département, ville...) constituent ce que l'on appelle le Distinguished Name (DN)

    [ v3_req ]
        # Définit les extensions X.509 de la requête
    subjectAltName = @alt_names
        # La liste SAN (noms alternatifs du serveur) sera définie dans la section [ alt_names ]

    [ alt_names ]
    DNS.1 = nginx.starfleet.lan
    DNS.2 = serveur.starfleet.lan
    IP.1 = 192.0.2.10
        # définis tous les noms/IP pour lesquels le certificat est valide
            # DNS.1 = nginx.starfleet.lan → le certificat est valide si le client se connecte à nginx.starfleet.lan
            # DNS.2 = serveur.starfleet.lan → valide aussi pour ce deuxième nom DNS
            # IP.1 = 192.0.2.10 → valide aussi si le client se connecte directement via cette IP
```


VERSION COPIABLE :

[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[ req_distinguished_name ]
C = FR
ST = Île-de-France
L = Paris
O = MaSociete
CN = ftp.starfleet.lan

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = nginx.starfleet.lan
DNS.2 = serveur.starfleet.lan
IP.1 = 192.168.1.1


### Génération d'une clé privée + certificat auto-signé :

```bash
sudo openssl req -x509 -nodes -newkey rsa:4096 \
    # openssl req : lance la génération d’une requête de signature (CSR) — ici utilisé pour créer le certificat auto-signé
    # -x509 : demande la génération directe d’un certificat auto-signé au lieu d’une CSR uniquement
    # -nodes : ne chiffrera pas la clé privée (no DES) → la clé privée n’a pas de passphrase. 
        # Important : nginx ne peut pas demander une passphrase au démarrage, donc la clé doit être sans passphrase. La passphrase est un mdp qui permet
        # d'acceder au fichier au se trouve la clé privée
    # -newkey rsa:4096 : crée une nouvelle paire clé RSA de 4096 bits (taille recommandée pour sécurité).

    -keyout /etc/ssl/private/nginx.key \
        # -keyout /etc/ssl/private/starfleet.lan.key : chemin où sera écrit la clé privée. Par convention les clés privées vont dans /etc/ssl/private

    -out /etc/ssl/certs/nginx.crt \
        # -out /etc/ssl/certs/nginx.crt : chemin du certificat X.509 auto-signé généré

    -days 3650 \
        # -days 3650 : durée de validité du certificat en jours (ici 10 ans). Pour tests/intranet, ok ; pour production pense à renouveler régulièrement

    -config /tmp/nginx_openssl.cnf \
        # Indique le fichier de configuration du certificat

    -extensions v3_req
        # indique d’appliquer la section v3_req (où subjectAltName pointe vers alt_names)
```

    
VERSION COPIABLE :

sudo openssl req -x509 -nodes -newkey rsa:4096 \
    -keyout /etc/ssl/private/nginx.key \
    -out /etc/ssl/certs/nginx.crt \
    -days 3650 \
    -config /tmp/nginx_openssl.cnf \
    -extensions v3_req



### Vérifications :

```bash
openssl x509 -in /etc/ssl/certs/nginx.crt -noout -text | grep -A2 "Subject Alternative Name"
    # Cette ligne vérifie que SAN est bien inclu
    # openssl x509 Utilise la sous-commande x509 d’OpenSSL : permet de lire, afficher ou convertir un certificat X.509
    # -in /etc/ssl/certs/nginx.crt : Chemin du certificat à analyser
    # -noout : Demande à OpenSSL de ne pas afficher le certificat encodé en Base64 (PEM).
        # Sans cette option, il y aurait aussi tout le bloc -----BEGIN CERTIFICATE----- ... -----END CERTIFICATE----- affiché → pas utile pour la lecture.
    # -text : Demande à OpenSSL d’afficher les informations du certificat en clair (lisibles par un humain) → DN, validité, extensions, SAN, etc
    # | grep -A2 "Subject Alternative Name" :
        # | : envoie la sortie de la commande précédente vers grep.
        # grep "Subject Alternative Name" : cherche la ligne contenant l’extension Subject Alternative Name (SAN).
        # -A2 : affiche aussi les 2 lignes suivantes (After = 2).
        # Utile car la liste des SAN (DNS, IP, etc.) est imprimée sur les lignes juste après.
```

    
### Permissions :

```bash
sudo chown root:root /etc/ssl/private/nginx.key
sudo chmod 600 /etc/ssl/private/nginx.key
sudo chmod 644 /etc/ssl/certs/nginx.crt
```


### Modifications du fichier de configuration du site :

```bash
nano /etc/nginx/sites-available/starfleet.lan
```


#### Rajout d'un block pour le HTTPS :
```bash
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name www.starfleet.lan starfleet.lan;

    root /var/www/starfleet.lan/html;
    index index.html index.htm;

    ssl_certificate /etc/ssl/certs/nginx.crt;
    ssl_certificate_key /etc/ssl/private/nginx.key;

    ssl_protocols TLSv1.2 TLSv1.3;
        # Nginx ne va accepter que TLS 1.2 et TLS 1.3 pour les connexions HTTPS
    ssl_ciphers 'HIGH:!aNULL:!MD5';
        # Définit la liste des algorithmes de chiffrement autorisés pour TLS.
            # HIGH → chiffrement fort (AES, ChaCha20…).
            # !aNULL → interdit les suites sans authentification (sécuriser la connexion).
            # !MD5 → interdit les suites utilisant MD5 (cryptographiquement cassé).
    ssl_prefer_server_ciphers on;
        # Le serveur choisit sa suite de chiffrement préférée, même si le client propose autre chose

    access_log /var/log/nginx/starfleet.lan.access.log;
    error_log  /var/log/nginx/starfleet.lan.error.log;
        # S'il y a plusieurs sites on peut aussi les mettre dans des fichiers différents

    location / {
        try_files $uri $uri/ =404;
    }
}
```


VERSION COPIABLE :

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name www.starfleet.lan starfleet.lan;
    root /var/www/starfleet.lan/html;
    index index.html index.htm;

    ssl_certificate /etc/ssl/certs/nginx.crt;
    ssl_certificate_key /etc/ssl/private/nginx.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'HIGH:!aNULL:!MD5';
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/starfleet.lan.access.log;
    error_log  /var/log/nginx/starfleet.lan.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}



#### Ajout d'une ligne dans le block HTTP pour redirection vers HTTPS :

```bash
    return 301 https://$host$request_uri;
        # return 301 → renvoie un code HTTP 301 (redirection permanente).
        # $host → nom de domaine demandé par le client (www.starfleet.lan ou starfleet.lan).
        # $request_uri → chemin et paramètres de la requête, par exemple /index.html?x=1.
        # Résultat : le navigateur est automatiquement redirigé vers la même URL mais en HTTPS.
```
    

### Modifications du fichier /etc/hosts sur la VM Web et la VM Client :

```bash
192.168.1.1   starfleet.lan www.starfleet.lan      
    # Ajout de cette ligne pour ne pas avoir la résolutions DNS d'un serveur DNS publique
```

### Activation et vérifications :

```bash
sudo nginx -t
sudo systemctl reload nginx
openssl s_client -connect www.starfleet.lan:443 -servername www.starfleet.lan
    # openssl s_client : C’est une commande OpenSSL pour se comporter comme un client TLS/SSL.
        # Elle se connecte à un serveur HTTPS ou FTPS et affiche toutes les informations de la session TLS : certificat, algorithmes de chiffrement, etc.
    # -connect www.starfleet.lan:443 : Indique l’adresse et le port du serveur à tester.
    # -servername www.starfleet.lan
        # Active SNI (Server Name Indication).
        # SNI permet à un serveur TLS de présenter le bon certificat lorsque plusieurs sites sont hébergés sur la même IP et le même port (virtual hosts HTTPS).
        # Sans SNI, le serveur pourrait renvoyer le certificat par défaut, qui ne correspond pas forcément au site demandé.
```

## Captures d'écran : 

Serveur web https fonctionnel :

![Serveur web https fonctionnel](/images/jobs/job06/Serveur_web_https.jpg)

