# MEMO RESEAU ET FIREWALL
## Projet Proxmox / pfSense

Guide de reference pour les concepts fondamentaux reseau et firewall.

---

## 1. IP vs Reseau (Base absolue)

### IP
- Identifie **UNE machine**
- Exemple : `192.168.30.10`
- Sert a se connecter (SSH, HTTPS, etc.)

### Reseau
- Identifie **UNE plage d'IP**
- Exemple : `192.168.30.0/24`
- Sert a regrouper et isoler

**Regle fondamentale :**
- Une IP appartient a un reseau
- Un reseau n'est PAS une machine

---

## 2. Meme reseau ou pas ?

Deux IP sont dans le meme reseau si :
- Elles ont le meme masque
- Leur partie reseau est identique

### Exemple 1 : Meme reseau (/24)
```
192.168.30.10
192.168.30.11
```
- Meme reseau
- Communication directe
- Pas besoin de firewall

### Exemple 2 : Reseaux differents
```
192.168.30.10
192.168.20.50
```
- Reseaux differents
- Firewall obligatoire

---

## 3. Pourquoi segmenter ?

**Objectif :** Forcer le passage par le firewall

```
LAN   -> 192.168.10.0/24
DMZ   -> 192.168.20.0/24
ADMIN -> 192.168.30.0/24
```

**Logique :**
1. Reseaux differents = routage obligatoire
2. Routage = filtrable
3. Filtrage = securite

---

## 4. Role du firewall (pfSense)

Le firewall est a la fois :
- **Gateway** (passerelle par defaut)
- **Routeur** (interconnecte les reseaux)
- **Filtre de securite** (applique les regles)

### Interfaces pfSense (exemple)
```
LAN   -> 192.168.10.1/24
DMZ   -> 192.168.20.1/24
ADMIN -> 192.168.30.1/24
WAN   -> IP publique
```

**A retenir :**
- Chaque interface definit un reseau
- Le firewall fait partie de chaque reseau (via son IP d'interface)

---

## 5. On ne se connecte JAMAIS a un reseau

**FAUX :**
- SSH vers `192.168.30.0/24`
- HTTPS vers "la DMZ"

**VRAI :**
- On se connecte a **UNE MACHINE**

### Exemples corrects
```bash
ssh admin@192.168.20.50      # Bastion
https://192.168.20.10        # NetBox
https://192.168.20.20        # Elasticsearch
```

---

## 6. Comment une communication fonctionne

### Scenario : ADMIN vers Bastion

1. PC admin (`192.168.30.10`) lance :
   ```bash
   ssh user@192.168.20.50
   ```

2. L'OS voit que la destination n'est pas locale

3. Il envoie au gateway `192.168.30.1`

4. Le firewall :
   - Regarde la destination
   - Applique la regle
   - Route vers la DMZ

5. Le retour est automatique (stateful)

---

## 7. Source / Destination : qui les definit ?

| Element          | Qui le definit                    |
|------------------|-----------------------------------|
| Source IP        | AUTOMATIQUE (IP de la machine)    |
| Destination IP   | Toi (quand tu tapes l'IP)         |
| Masque           | Config reseau                     |
| Regle de securite| Firewall                          |

**Important :** Tu ne choisis JAMAIS la source IP (elle est determinee automatiquement)

---

## 8. Ou definir la communication ?

**Uniquement sur le firewall**

### Exemple de regle pfSense
```
Interface   : ADMIN
Source      : 192.168.30.0/24
Destination : 192.168.20.50
Port        : 22
Action      : ALLOW
```

---

## 9. Erreurs classiques a eviter

- Tout mettre dans un seul /24
- Penser que l'interface remplace le reseau
- Croire qu'on "se connecte a un reseau"
- Mettre des regles sur les VMs au lieu du firewall
- Confondre IP et reseau dans les regles

---

## 10. Phrase cle (jury / oral)

> Les administrateurs se connectent aux hotes, pas aux reseaux. La segmentation en sous-reseaux distincts force le passage par le firewall, qui assure le routage et le filtrage inter-zones selon le principe du moindre privilege.

---

## Checklist mentale rapide

- [ ] IP = machine
- [ ] Reseau = plage
- [ ] Reseaux differents = firewall
- [ ] Une interface = une IP dans un reseau
- [ ] Regles = firewall
- [ ] Acces = IP d'une machine

---

## Annexe : Notations CIDR courantes

| Notation | Masque          | Nombre d'hotes |
|----------|-----------------|----------------|
| /32      | 255.255.255.255 | 1              |
| /24      | 255.255.255.0   | 254            |
| /16      | 255.255.0.0     | 65 534         |
| /8       | 255.0.0.0       | 16 777 214     |

---

## Annexe : Ports courants a connaitre

| Service       | Port | Protocole |
|---------------|------|-----------|
| SSH           | 22   | TCP       |
| HTTP          | 80   | TCP       |
| HTTPS         | 443  | TCP       |
| DNS           | 53   | UDP/TCP   |
| DHCP (client) | 68   | UDP       |
| DHCP (server) | 67   | UDP       |
| RDP           | 3389 | TCP       |
| ICMP (ping)   | -    | ICMP      |

---

## Annexe : Schema de flux type

```
[PC Admin]          [Firewall]           [Serveur DMZ]
192.168.30.10  -->  192.168.30.1    -->  192.168.20.50
                    (regle ADMIN)
                    192.168.20.1
```

Le firewall possede une IP dans chaque reseau qu'il interconnecte.

---

*Document de reference - Projet Infrastructure Proxmox/pfSense*
