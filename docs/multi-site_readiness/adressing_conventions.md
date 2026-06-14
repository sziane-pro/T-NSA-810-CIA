# Convention d’adressage IP

## Présentation
La segmentation réseau repose sur une séparation par **site** et par **segment fonctionnel**.
Les adresses utilisent le bloc privé `192.168.0.0/16`.

Chaque site dispose de trois segments principaux :

| Segment | Rôle                                                 | Code |
|---------|------------------------------------------------------|-----:|
| LAN     | Réseau utilisateurs                                  | `10` |
| DMZ     | Services exposés / bastion / services intermédiaires | `20` |
| ADMIN   | Administration, supervision et management            | `30` |

## Logique retenue

```text
192.168.<code_site + code_segment.0/24
```

Le troisième octet permet d’identifier à la fois le **site** et le **segment réseau**.

- Site 1 : base `0` → LAN `10`, DMZ `20`, ADMIN `30`
- Site 2 : base `100` → LAN `110`, DMZ `120`, ADMIN `130`
- Site 3 : base `200` → LAN `210`, DMZ `220`, ADMIN `230`
note : il n'y a pas nécessairement besoin d'avoir les 3 réseaux nommés ainsi dans chaque site. Cependant, s'ils sont présent, ils devront respectés cette convetion d'adressage.

Exemple :

| Site   | Segment | Plage IP           | Usage              |
|--------|---------|--------------------|--------------------|
| Site 1 | LAN     | `192.168.10.0/24`  | Utilisateurs       |
| Site 1 | DMZ     | `192.168.20.0/24`  | Services exposés   |
| Site 1 | ADMIN   | `192.168.30.0/24`  | Management         |
| Site 2 | LAN     | `192.168.110.0/24` | Utilisateurs       |
| Site 2 | DMZ     | `192.168.120.0/24` | Bastion + Services |
| Site 2 | ADMIN   | `192.168.130.0/24` | Management         |

Cette convention permet une lecture rapide des plages IP : le site et le type de réseau sont identifiables directement dans l’adresse.

**Limite :** cette logique est adaptée aux premiers sites comme demandés dans l'énoncé, mais le troisième octet d’une adresse IPv4 ne pouvant pas dépasser `255`, elle devra évoluer vers un plan plus scalable en cas d’ajout de nombreux sites.