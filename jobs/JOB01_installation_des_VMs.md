# JOB01 - Installation des machines virtuelles


Hyperviseur de type 2 utilisé : VMware Workstation


## Commandes pour afficher les informations du hardware recherchées : 

```bash
echo "Noyau: $(uname -r)"
echo "RAM: $(free -h | awk '/Mem:/ {print $2}')"
echo "Stockage:"; lsblk -d -o NAME,SIZE,MODEL | tail -n +2
echo "Cartes réseau:"; lspci | grep -Ei 'network|ethernet|wireless|wi-fi'
echo "Cœurs CPU: $(nproc)"
```

## Captures d'écran :


Matériel installé pour la VM serveur :
![Materiel VM serveur](/images/jobs/job01/materiel_vm_serveur.jpg)

Matériel installé pour la VM cliente :
