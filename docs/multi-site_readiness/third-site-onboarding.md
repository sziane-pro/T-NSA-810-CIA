## Onboarding of a new Site

### 1. Définir la segmentation réseau du troisième site

Créer trois sous-réseaux distincts pour le site x, en conservant la même logique que les sites existants(cf. [adressing_conventions.md](adressing-conventions.md))

| Segment | Plage IP           | Usage                                                |
|---------|--------------------|------------------------------------------------------|
| LAN     | `192.168.210.0/24` | Réseau utilisateurs                                  |
| DMZ     | `192.168.220.0/24` | Services exposés, bastion ou services intermédiaires |
| ADMIN   | `192.168.230.0/24` | Administration, supervision et management            |

Cette segmentation permet d’isoler les utilisateurs, les services exposés et les accès d’administration.

### 2. Configurer les équipements réseau du nouveau site 

Mettre en place les équipements nécessaires au fonctionnement du site : routeur, pare-feu, switchs et points d’accès si besoin.

Chaque équipement doit être configuré avec les VLANs ou interfaces correspondant avec les segments réseau souhaités.

### 3. Mettre en place le routage entre les sites

Configurer le routage afin de permettre la communication entre le site x et les sites existants.

Le routage doit permettre uniquement les échanges nécessaires entre les sites, par exemple l’accès aux services internes, aux outils d’administration ou aux ressources partagées.

### 4. Appliquer les règles de pare-feu

Définir les règles de filtrage entre les différents segments réseau.

Par défaut, les flux doivent être bloqués, puis uniquement les flux nécessaires doivent être autorisés. Cette approche permet de respecter le principe du moindre privilège.

Exemples de règles :

* Le LAN du site x peut accéder aux services internes autorisés.
* La DMZ reste isolée du LAN sauf exception contrôlée.
* Le réseau ADMIN est réservé aux accès d’administration et de supervision.
* Les flux entre sites sont limités aux besoins métier.

### 5. Intégrer le site x aux services d’infrastructure

Configurer les services nécessaires au bon fonctionnement du site x : DNS, DHCP, annuaire, supervision, journalisation et sauvegarde si besoin.

Cette étape permet aux postes utilisateurs et aux serveurs du site x d’utiliser les mêmes services centraux que les autres sites.

### 6. Déployer les services dans la DMZ si nécessaire

Installer ou raccorder les services exposés du site x dans le segment DMZ.

La DMZ peut accueillir un bastion, un reverse proxy ou des services devant être accessibles depuis d’autres réseaux, tout en restant séparée du LAN et du réseau ADMIN.

### 7. Intégrer le réseau ADMIN à la supervision

Ajouter les équipements et serveurs du site x aux outils de supervision et de management.

Cela permet de surveiller l’état du réseau, la disponibilité des services, les performances et les éventuelles alertes de sécurité.

### 8. Tester la connectivité réseau

Vérifier que chaque segment fonctionne correctement.

Les tests doivent inclure :

* la communication entre les postes du LAN et les services autorisés ;
* l’accès aux services de la DMZ ;
* l’accès d’administration depuis le réseau ADMIN ;
* la résolution DNS ;
* le routage entre les sites ;
* la remontée des logs et des alertes.

### 9. Valider les règles de sécurité

Contrôler que les flux non autorisés sont bien bloqués.

Cette étape permet de vérifier que le cloisonnement entre LAN, DMZ et ADMIN est respecté, et que le site x ne crée pas de faille dans l’architecture existante.

### 10. Documenter l’intégration du site x

Mettre à jour la documentation réseau avec les nouvelles plages IP, les règles de pare-feu, les équipements ajoutés et les services configurés.

Cette documentation permet de faciliter la maintenance, le support et les futures évolutions de l’infrastructure.
