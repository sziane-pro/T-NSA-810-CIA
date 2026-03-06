## Setup de la VM1

### Vérification de l'adresse IP de la VM

Permet de vérifier les interfaces réseau et l'adresse IP attribuée :

``` bash
ip a
```

------------------------------------------------------------------------

### Modification du VLAN via Netplan

Le VLAN est configuré directement dans la configuration réseau Ubuntu.

Édition du fichier Netplan :

``` bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

------------------------------------------------------------------------

### Exemple de configuration Netplan

``` yaml
network:
  version: 2
  ethernets:
    enp6s18: {}

  vlans:
    enp6s18.30:
      id: 30
      link: enp6s18
      addresses:
        - 192.168.30.10/24
      gateway4: 192.168.30.1
      nameservers:
        addresses: [8.8.8.8]
```

### Explication

-   `enp6s18` : interface réseau physique
-   `enp6s18.30` : interface VLAN
-   `id: 30` : identifiant du VLAN
-   `link: enp6s18` : interface parent utilisée pour le VLAN

------------------------------------------------------------------------

### Application de la configuration

Une fois les modifications effectuées :

``` bash
sudo netplan apply
```

------------------------------------------------------------------------

### Vérification des interfaces réseau

Lister les interfaces réseau :

``` bash
ip link
```

ou

``` bash
ip a
```

------------------------------------------------------------------------

### Suppression d'une ancienne interface VLAN

Si une ancienne interface VLAN apparaît encore dans la liste (ex :
`enp6s18.10`) :

``` bash
sudo ip link delete enp6s18.10
```

Cela permet de supprimer l'interface VLAN active qui n'est plus utilisée
dans la configuration Netplan.

------------------------------------------------------------------------

### Vérification finale

Vérifier que seule l'interface VLAN correcte est présente :

``` bash
ip a
```

Interface attendue :

    enp6s18.30
