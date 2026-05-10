# Mise en place d'un parefeu 


## Intallation de ufw :

```bash
apt install ufw -y
ufw enable
```

## Récapitulatif des connexions requises par job :

JOB01 : port 22 ouvert en entrant de ma machine hôte 192.168.159.1 (facilité d'administration)
JOB03 : port 67 ouvert en sortant en udp pour le serveur DHCP
JOB05 : port 53 ouvert en entrant udp tcp pour le serveur DNS
JOB06 : ports 80 et 443 ouverts entrant sortant tcp pour le serveur web
JOB07 : port 21 ouvert en entrant en tcp + la plage passive FTPS 30000-30100


## Commandes :

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow in on ens34 to any port 67 proto udp
ufw allow in on ens34 from 192.168.1.0/24 to any port 53 proto udp
ufw allow in on ens34 from 192.168.1.0/24 to any port 53 proto tcp
ufw allow in on ens34 from 192.168.1.0/24 to any port 80 proto tcp
ufw allow in on ens34 from 192.168.1.0/24 to any port 443 proto tcp
ufw allow in on ens34 from 192.168.1.0/24 to any port 21 proto tcp
ufw allow in on ens34 from 192.168.1.0/24 to any port 30000:31000 proto tcp
ufw allow in on ens33 from 192.168.159.1/24 to any port 22 proto tcp
```

## capture d'écran :

![Règles mises en application](/images/jobs/job12/regles_ufw.jpg)

