# JOB07 - Installation et configuration du serveur FTP


## Explications :

Voir mon cours : https://github.com/Teva-Clairefond/Cours-generalites/blob/main/Cours/Linux%20generals/Installation%20d'un%20serveur%20FTP.md


## Installation du serveur FTPS :


```bash
sudo apt install vsftpd -y

sudo nano /etc/vsftpd.conf
    listen=YES
        # Le serveur écoute les IPv4 
    anonymous_enable=NO
        # Ne permet pas aux utilisateurs anonymes de se connecter au serveur
    local_enable=YES
        # Autoriser les utilisateurs locaux à accéder au serveur ftp avec leur compte d'OS, sans compte virtuel
    write_enable=YES
        # Autoriser les opérations d’écriture (upload, création de dossier, suppression)
    chroot_local_user=YES
        # chroot (Change Root) = Quand un utilisateur est “chrooté” dans un répertoire, ce répertoire devient la nouvelle racine (/) pour cet utilisateur.
        # L’utilisateur ne peut pas remonter plus haut que ce répertoire. Il ne voit pas le reste du système de fichiers.
    allow_writeable_chroot=YES
        # Permet aux utilisateurs d'uploader et gérer leurs fichiers dans leur home directory, mais peut présenter des failles de sécurité.

```

## Passage à FTPS :


### Activer le FTPS dans le fichier de config FTP :

Nous avons déjà créé le certificat dans le job précédent donc il faut le réutilisé :

```bash
nano /etc/vsftpd.conf
    # A remplacer à la fin du fichier
    rsa_cert_file=/etc/ssl/certs/nginx.crt
        # Chemin vers le certificat X.509 du serveur
    rsa_private_key_file=/etc/ssl/private/nginx.key
        # Chemin vers la clé privée RSA du serveur
    ssl_enable=YES
        # Active TLS
```

### Vérification 2 :

```bash
systemctl status vsftpd

sudo ss -tuln | grep :21
    # Vérifie que le service écoute bien sur le port 21
openssl s_client -connect 192.168.1.1:21 -starttls ftp
    # Test la connexion 
```

## Création d'un utilisateur dont le répertoire de travail est chrooté sur le dossier :

```bash
sudo useradd -d /var/www/starfleet.lan/html -s /bin/bash ftpweb
    # Création de l'utilisateur ftpweb dont le répertoire principal est /var/www/starfleet.lan/html avec le shell bash
sudo passwd ftpweb
    # Permet de définir un mot de passe à l'utilisateur

sudo chown -R ftpweb:www-data /var/www/starfleet.lan/html
sudo chmod -R 775 /var/www/starfleet.lan/html
    # Attribution de droit sur le dossier web

nano /etc/vsftpd.conf
    # Ajouter :
    local_root=/var/www/starfleet.lan/html

sudo systemctl restart vsftpd
```

### Vérification :

Nous allons nous connecter en ftps depuis la machine cliente et allons vérifier le chroot :

```bash
lftp -u ftpweb ftp.starfleet.lan

# Le certificat étant autosigné, il est par défaut non-accepté. Il faut donc forcer son acceptation dans le cadre de notre projet :
set ssl:verify-certificate no
```


## Tests finaux et capture d'écran :


Installation d'un client ftps : 

```bash
sudo apt install lftp
```

Connexion SSL/TLS fonctionnelle :

![Connexion SSL/TLS fonctionnelle](/images/jobs/job07/Connexion_TLS.jpg)


CHROOT fonctionnel sur le répertoire web :

![Chroot fonctionnel](/images/jobs/job07/Chroot_fonctionnel.jpg)