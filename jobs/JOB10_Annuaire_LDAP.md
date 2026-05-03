# Mise en place d'un annuaire LDAP :


## Explications :


## Installation et configuration :


### Installation des paquets :

```bash
apt-get update
apt-get install slapd ldap-utils
apt-get install phpldapadmin
apt install php8.4-ldap -y
systemctl restart php8.4-fpm
```
Please enter the password for the admin entry in your LDAP directory. Administrator password: -> Mettre un mot de passe.


```bash
sudo dpkg-reconfigure slapd
```

Omit OpenLDAP server configuration? → No
DNS domain name → starfleet.lan
Organization name → Starfleet
Administrator password → 123
Remove database when slapd is purged? → No
Move old database? → Yes


### Configuration de ldap :

```bash
nano /etc/ldap/ldap.conf
    BASE    dc=starfleet,dc=lan
    URI     ldap://127.0.0.1
```

### Création de l'arborescence ldap :

```bash
nano /root/base-ldap.ldif
    dn: ou=people,dc=starfleet,dc=lan
    objectClass: organizationalUnit
    ou: people

    dn: ou=groups,dc=starfleet,dc=lan
    objectClass: organizationalUnit
    ou: groups


ldapadd -x -D "cn=admin,dc=starfleet,dc=lan" -W -f /root/base-ldap.ldif
```

### Installation et configuration de l'interface web phpldapadmin :


### Configuration de phpldapadmin :

```bash
nano /etc/phpldapadmin/config.php
    # Remplacer : $servers->setValue('server','base',array('dc=example,dc=com'));
    # Par :
    $servers->setValue('server','base',array('dc=starfleet,dc=lan'));
```

#### Configuration DNS :

```bash
nano /etc/bind/db.starfleet.lan
    ...
    ldap     IN     A     192.168.1.1
```
Augmenter le Serial

```bash
systemctl reload bind9
```

#### Création du site :

```bash
nano /etc/nginx/sites-available/ldap.starfleet.lan
    server {
        listen 80;
        listen [::]:80;

        server_name ldap.starfleet.lan;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name ldap.starfleet.lan;

        root /usr/share/phpldapadmin/htdocs;
        index index.php index.html;

        ssl_certificate     /etc/ssl/certs/nginx.crt;
        ssl_certificate_key /etc/ssl/private/nginx.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'HIGH:!aNULL:!MD5';

        access_log /var/log/nginx/ldap.starfleet.lan.access.log;
        error_log  /var/log/nginx/ldap.starfleet.lan.error.log;

        # Limiter l'accès au LAN
        allow 192.168.1.0/24;
        deny all;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        }

        location ~ /\.ht {
            deny all;
        }
    }
```

Activation du site en créant le lien symbolique :

```bash
ln -s /etc/nginx/sites-available/ldap.starfleet.lan /etc/nginx/sites-enabled/
```

#### Vérifiaction :

```bash
nginx -t
systemctl restart php8.4-fpm
systemctl reload nginx
```

Aller sur la machine cliente :
https://ldap.starfleet.lan


## Création d'utilisateurs mise en place de l'authentification web :


### Création des utilisateurs :


Une fois connecté au site de phpldapadmin, nous allons créer cette arborescence :

    dc=starfleet,dc=lan         ← Racine du domaine LDAP
    ├─ ou=people                        
    │   └─ cn=Jean Dupont            
    └─ ou=groups
        └─ cn=webusers      

dc = starfleet > Create a child entry > organisationnal unit > *Choisir un nom =* people > Create object > Commit
dc = starfleet > Create a child entry > organisationnal unit > *Choisir un nom =* groups > Create object > Commit

dc = starfleet > View 2 children > people > Create a child entry > Default > InetOrgPerson > Proceed 
    > RDN > User Name (uid)
    > cn > Jean Dupont
    > sn > Dupont
    > mail > jeandupont@gmail.com
    > ou > people
    > mdp > 123
    > User Name > Dupont
    > Create Object > Commit

Créer les utilisateurs via InetOrgPerson est plus simple dans le cadre de ce tp.


dc = starfleet > View 2 children > groups > Create a child entry > Default > groupOfNames > Proceed 
    > RDN > cn
    > cn > webusers
    > member > uid=Dupont ou=people dc=starfleet dc=lan
    > ou > groups
    > Create Object > Commit


### Mise en place de l'authentification :

```bash
mkdir -p /var/www/auth
nano /var/www/auth/ldap_auth.php
    <?php

    function deny(): void {
        header('WWW-Authenticate: Basic realm="Starfleet LDAP"');
        http_response_code(401);
        exit;
    }

    $auth = $_SERVER['HTTP_AUTHORIZATION'] ?? '';

    if (!str_starts_with($auth, 'Basic ')) {
        deny();
    }

    $decoded = base64_decode(substr($auth, 6), true);

    if ($decoded === false || !str_contains($decoded, ':')) {
        deny();
    }

    [$username, $password] = explode(':', $decoded, 2);

    if ($username === '' || $password === '') {
        deny();
    }

    $ldapHost = 'ldap://127.0.0.1';
    $baseDn   = 'dc=starfleet,dc=lan';

    $userDn = 'uid=' . ldap_escape($username, '', LDAP_ESCAPE_DN) . ',ou=people,' . $baseDn;

    $ldap = ldap_connect($ldapHost);

    if (!$ldap) {
        deny();
    }

    ldap_set_option($ldap, LDAP_OPT_PROTOCOL_VERSION, 3);
    ldap_set_option($ldap, LDAP_OPT_REFERRALS, 0);

    /*
    * Authentification LDAP :
    * l'utilisateur doit exister et le mot de passe doit être correct.
    */
    if (!@ldap_bind($ldap, $userDn, $password)) {
        deny();
    }

    /*
    * Vérification que l'utilisateur appartient au groupe webusers.
    */
    $groupsOu = 'ou=groups,' . $baseDn;
    $escapedUserDn = ldap_escape($userDn, '', LDAP_ESCAPE_FILTER);

    $filter = "(&(objectClass=groupOfNames)(cn=webusers)(member={$escapedUserDn}))";

    $search = @ldap_search($ldap, $groupsOu, $filter, ['cn']);

    if (!$search) {
        deny();
    }

    $entries = ldap_get_entries($ldap, $search);

    if ($entries['count'] < 1) {
        deny();
    }

    /*
    * Authentification réussie.
    */
    http_response_code(204);
    exit;
```

Permissions : 

```bash
chown -R www-data:www-data /var/www/auth
chmod 750 /var/www/auth
chmod 640 /var/www/auth/ldap_auth.php
```

Création d'un snippet (configuration réutilisable) nginx :

```bash
nano /etc/nginx/snippets/ldap-auth.conf
    auth_request /_ldap_auth;

    location = /_ldap_auth {
        internal;
        auth_request off;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /var/www/auth/ldap_auth.php;
        fastcgi_param SCRIPT_NAME /_ldap_auth;
        fastcgi_param HTTP_AUTHORIZATION $http_authorization;

        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
```

Ajout dans les paramètres du site :

```bash
nano /etc/nginx/sites-available/www8.starfleet.lan
    server {
    listen 443 ssl;
    ...
    include snippets/ldap-auth.conf;
    ...
    }
```


Application :

```bash
nginx -t
systemctl reload nginx
```

### Vérification :

En ce rendant sur https://www8.starfleet.lan


## Captures d'écran :

![phpldapadmin fonctionnel](/images/jobs/job10/phpldapadmin%20fonctionnel.jpg)

![authentification fonctionnelle](/images/jobs/job10/authentification.jpg)