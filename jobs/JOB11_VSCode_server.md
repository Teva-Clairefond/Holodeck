# Visual Studio Code Server


## Installation :

```bash
apt update
curl -fsSL https://code-server.dev/install.sh | sh -s -- --dry-run
curl -fsSL https://code-server.dev/install.sh | sh
```

## Création d'un utilisateur pour VScore :


Executer code-server nécessite un UID et d'avoir des droits sur les fichiers. Il faut donc créer un utilisateur pour VScore :

```bash
useradd -m -s /bin/bash vscore
passwd vscore
```

```bash
usermod -aG www-data vscore
chown -R www-data:www-data /var/www
chmod -R 775 /var/www
```


## Désactivation de l’authentification interne de code-server :

Code-server utilise normalement une authentification interne par mot de passe, mais dans votre cas on va la désactiver et laisser Nginx + LDAP gérer l’accès :

```bash
su - vscore
mkdir -p ~/.config/code-server
nano ~/.config/code-server/config.yaml
    bind-addr: 127.0.0.1:8080
        # code-server écoute seulement en local
    auth: none
        # code-server ne demande pas de mot de passe
    cert: false
        # HTTPS sera géré par Nginx

systemctl restart code-server@vscore
```

## Création d'un script d’authentification LDAP pour vsusers :

```bash
mkdir -p /var/www/auth
nano /var/www/auth/ldap_auth_vsusers.php
    <?php

    function deny(): void {
        header('WWW-Authenticate: Basic realm="Starfleet VSCode"');
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

    if (!@ldap_bind($ldap, $userDn, $password)) {
        deny();
    }

    $groupsOu = 'ou=groups,' . $baseDn;
    $escapedUserDn = ldap_escape($userDn, '', LDAP_ESCAPE_FILTER);

    $filter = "(&(objectClass=groupOfNames)(cn=vsusers)(member={$escapedUserDn}))";

    $search = @ldap_search($ldap, $groupsOu, $filter, ['cn']);

    if (!$search) {
        deny();
    }

    $entries = ldap_get_entries($ldap, $search);

    if ($entries['count'] < 1) {
        deny();
    }

    http_response_code(204);
    exit;
```

```bash
chown -R www-data:www-data /var/www/auth
chmod 750 /var/www/auth
chmod 640 /var/www/auth/ldap_auth_vsusers.php
```


## Création du site :

```bash
nano /etc/nginx/sites-available/vscore.starfleet.lan
    server {
        listen 80;
        listen [::]:80;

        server_name vscore.starfleet.lan;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name vscore.starfleet.lan;

        ssl_certificate     /etc/ssl/certs/nginx.crt;
        ssl_certificate_key /etc/ssl/private/nginx.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'HIGH:!aNULL:!MD5';

        access_log /var/log/nginx/vscore.starfleet.lan.access.log;
        error_log  /var/log/nginx/vscore.starfleet.lan.error.log;

        location = /_ldap_auth_vsusers {
            internal;

            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME /var/www/auth/ldap_auth_vsusers.php;
            fastcgi_param SCRIPT_NAME /_ldap_auth_vsusers;
            fastcgi_param HTTP_AUTHORIZATION $http_authorization;

            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        }

        location / {
            auth_request /_ldap_auth_vsusers;

            proxy_pass http://127.0.0.1:8080;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            proxy_set_header Authorization "";
        }
    }
```

```bash
ln -s /etc/nginx/sites-available/vscore.starfleet.lan /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```


## Ajout du DNS :

```bash
nano /etc/bind/db.starfleet.lan
    ...
    vscore  IN  A  192.168.1.1
    ...
```

Incrémenter le Serial.

```bash
named-checkzone starfleet.lan /etc/bind/db.starfleet.lan
systemctl reload bind9
```

## Création d'un groupe d'utilisateurs pour vsusers dans phpldapadmin :


dc = starfleet > View 2 children > groups > Create a child entry > Default > groupOfNames > Proceed 
    > RDN > cn
    > cn > vsusers
    > member > uid=Dupont ou=people dc=starfleet dc=lan
    > ou > groups
    > Create Object > Commit



## Capture d'écran :

![vscore fonctionnel](/images/jobs/job11/vscore_fonctionnel.jpg)