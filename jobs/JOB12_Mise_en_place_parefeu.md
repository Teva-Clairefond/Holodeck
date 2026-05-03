# JOB12 - Mise en place du pare-feu


## Explications :

L'objectif est d'autoriser uniquement les flux nécessaires au projet Holodeck.

La VM serveur possède deux cartes réseau :

- une carte WAN pour sortir vers Internet ;
- une carte LAN pour les services du domaine `starfleet.lan`.

Dans les jobs précédents, l'interface LAN utilisée est `ens34` avec l'adresse
`192.168.1.1/24`.

Le pare-feu sera mis en place avec `nftables`, qui est le pare-feu moderne
utilisé par Debian.


## Ports autorisés :

| Service | Port | Protocole | Interface | Rôle |
| --- | --- | --- | --- | --- |
| DHCP | 67 | UDP | LAN | Attribution IP à la VM cliente |
| DHCP client | 68 | UDP | WAN | Récupération IP de la carte WAN |
| DNS | 53 | UDP/TCP | LAN | Résolution du domaine `starfleet.lan` |
| HTTP | 80 | TCP | LAN | Redirection vers HTTPS |
| HTTPS | 443 | TCP | LAN | Sites web, phpMyAdmin, LDAP web, Cockpit, VSCode |
| FTPS | 21 | TCP | LAN | Connexion FTP sécurisée |
| FTPS passif | 40000-40010 | TCP | LAN | Transfert de fichiers FTP |
| ICMP | ping | ICMP | LAN | Tests réseau |

Les ports suivants ne doivent pas être ouverts directement :

- `3306` : MariaDB reste local au serveur ;
- `389` et `636` : LDAP est utilisé localement par les scripts PHP ;
- `8080` : VS Code Server écoute uniquement sur `127.0.0.1` ;
- `9090` : Cockpit est accessible via Nginx sur `admin.starfleet.lan` ;
- `22` : SSH n'est pas nécessaire si l'administration passe par Cockpit.


## Vérification des interfaces :

Dans cette procédure :

- `ens33` correspond à la carte WAN ;
- `ens34` correspond à la carte LAN ;
- `192.168.1.0/24` correspond au réseau LAN ;
- `192.168.1.1` correspond à l'adresse LAN du serveur.


## Installation de nftables :

En root :

```bash
apt update
apt install nftables -y
systemctl enable nftables
```


## Configuration des ports passifs du serveur FTP :

Le FTP utilise un port de contrôle, le port `21`, mais aussi des ports de
données. Pour éviter d'ouvrir une grande plage de ports, on force une petite
plage passive.

```bash
nano /etc/vsftpd.conf
    pasv_enable=YES
    pasv_min_port=40000
    pasv_max_port=40010
    pasv_address=192.168.1.1

    force_local_logins_ssl=YES
    force_local_data_ssl=YES
```

Redémarrer le service FTP :

```bash
systemctl restart vsftpd
```


## Création des règles nftables :

```bash
nano /etc/nftables.conf
```

Contenu du fichier :

```bash
#!/usr/sbin/nft -f

flush ruleset

define wan_if = "ens33"
define lan_if = "ens34"
define lan_net = 192.168.1.0/24
define ftp_passive_ports = 40000-40010

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state invalid drop
        ct state established,related accept
        iifname "lo" accept

        # DHCP client côté WAN si la carte WAN obtient son IP automatiquement
        iifname $wan_if udp sport 67 udp dport 68 accept

        # DHCP pour la VM cliente.
        # Ne pas filtrer sur ip saddr ici, car une première demande DHCP peut venir de 0.0.0.0.
        iifname $lan_if udp sport 68 udp dport 67 accept

        # DNS local starfleet.lan
        iifname $lan_if ip saddr $lan_net udp dport 53 accept
        iifname $lan_if ip saddr $lan_net tcp dport 53 accept

        # Nginx : sites web, phpMyAdmin, admin, LDAP web et VSCode
        iifname $lan_if ip saddr $lan_net tcp dport { 80, 443 } accept

        # FTPS
        iifname $lan_if ip saddr $lan_net tcp dport 21 accept
        iifname $lan_if ip saddr $lan_net tcp dport $ftp_passive_ports accept

        # Tests de connectivité depuis le LAN
        iifname $lan_if ip saddr $lan_net ip protocol icmp accept

        counter drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;

        ct state invalid drop
        ct state established,related accept

        # Autoriser uniquement les sorties utiles de la VM cliente vers Internet
        iifname $lan_if oifname $wan_if ip saddr $lan_net tcp dport { 80, 443 } accept
        iifname $lan_if oifname $wan_if ip saddr $lan_net udp dport { 53, 123 } accept

        counter drop
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # NAT pour permettre à la VM cliente de sortir par la carte WAN
        oifname $wan_if ip saddr $lan_net masquerade
    }
}
```


## Explication des règles :

```bash
ct state established,related accept
    # Autorise les réponses aux connexions déjà établies.

iifname "lo" accept
    # Autorise les échanges internes à la machine. C'est nécessaire pour PHP-FPM,
    #LDAP local, Cockpit via Nginx et VS Code Server.

iifname $lan_if ip saddr $lan_net tcp dport { 80, 443 } accept
    # Autorise les sites web uniquement depuis le LAN.

iifname $lan_if ip saddr $lan_net tcp dport $ftp_passive_ports accept
    # Autorise uniquement la plage passive FTPS définie dans `vsftpd.conf`.

type filter hook input priority 0; policy drop;
    # Tout ce qui n'est pas explicitement autorisé est refusé.
```



## Vérification de la configuration :

```bash
nft -c -f /etc/nftables.conf
systemctl restart nftables
nft list ruleset
```


## Vérifications depuis la VM serveur :

Voir les ports en écoute :

```bash
ss -tulpen
```

Les ports publics attendus sont :

- `53` pour DNS ;
- `67` pour DHCP ;
- `80` et `443` pour Nginx ;
- `21` pour FTPS ;
- `40000-40010` pour les transferts FTPS passifs.

Les ports `8080`, `9090`, `3306` et `389` peuvent exister en local, mais ils ne
doivent pas être ouverts directement sur le LAN ou sur le WAN.


## Vérifications depuis la VM cliente :

Tester le DNS :

```bash
nslookup www8.starfleet.lan 192.168.1.1
nslookup php.starfleet.lan 192.168.1.1
nslookup admin.starfleet.lan 192.168.1.1
nslookup vscore.starfleet.lan 192.168.1.1
```

Tester les sites HTTPS :

```bash
curl -k https://www8.starfleet.lan
curl -k https://php.starfleet.lan
curl -k https://admin.starfleet.lan
curl -k https://vscore.starfleet.lan
```

Tester le FTPS :

```bash
lftp -u ftpweb ftp.starfleet.lan
set ssl:verify-certificate no
ls
```

Tester que les ports internes ne sont pas accessibles :

```bash
nc -vz 192.168.1.1 3306
nc -vz 192.168.1.1 389
nc -vz 192.168.1.1 8080
nc -vz 192.168.1.1 9090
```

Ces tests doivent échouer depuis la VM cliente.


## Sauvegarde des règles :

La configuration est stockée dans :

```bash
/etc/nftables.conf
```

