# JOB02 - Pas de compte sudo 

## Explications :

Nous allons modifier le fichier sudoers afin de donner à notre utilisateur les droits sudo uniquement sur certaines commandes.
De plus nous allons modifier le fichier .bashrc pour que notre utilisateur n'ait pas à marquer "sudo".

## Modification du fichier sudoers :

En root : 

```bash
su -
cd /etc/sudoers
    
    teva    ALL=(ALL:ALL)   /usr/bin/apt install *, /usr/bin/systemctl *, /usr/sbin/ip *, /usr/bin/nano, /etc/default/isc-dhcp-server, /usr/bin/chown, /usr/bin/chmod, /usr/bin/named-checkconf

su - teva
```

## Modification du fichier .bashrc :

```bash
nano .bashrc
    ...
    alias apt="sudo apt"
    alias systemctl="sudo systemctl"
    alias ip="sudo ip"
    alias nano="sudo nano"
    alias chown="sudo chown"
    alias chmod="sudo chmod"
    alias named-checkconf="sudo named-checkconf"

source ~/.bashrc
```

## Capture d'écran :
![Pas de compte sudo](/images/jobs/job02/pas_de_groupe_sudo.jpg)

On peut voir ici que l'utilisateur ne fait pas partie du groupe sudo, et qu'il n'utilise pas "sudo" dans sa commande.

