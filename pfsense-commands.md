# pfSense - Commandes Utiles

## Configuration Clavier

### Changer le clavier en AZERTY (temporaire)
Depuis le menu principal, sélectionner **8) Shell**, puis :
```sh
kbdcontrol -l /usr/share/vt/keymaps/fr.kbd
```

### Rendre le changement permanent
```sh
echo 'keymap="fr"' >> /etc/rc.conf.local
```

### Lister les keymaps disponibles
```sh
ls /usr/share/vt/keymaps/ | grep fr
```

Résultat :
- `fr.acc.kbd` - Français avec accents
- `fr.dvorak.acc.kbd` - Français Dvorak avec accents
- `fr.kbd` - Français standard
- `fr.bepo.kbd` - Français BÉPO
- `fr.dvorak.kbd` - Français Dvorak
- `fr.macbook.kbd` - Français MacBook

---

## Configuration Réseau

### Assigner les interfaces
Menu principal → **1) Assign Interfaces**

### Configurer les VLANs
Menu principal → **1) Assign Interfaces** → Répondre **y** à "Should VLANs be set up now?"

Configuration recommandée pour Site 1 :
- VLAN 10 → LAN (`192.168.10.0/24`)
- VLAN 20 → DMZ (`192.168.20.0/24`)
- VLAN 30 → ADMIN (`192.168.30.0/24`)

### Configurer une IP sur une interface
Menu principal → **2) Set interface(s) IP address**

---

## Gestion Système

### Accéder au shell
Menu principal → **8) Shell**

### Redémarrer le système
Menu principal → **5) Reboot system**

### Arrêter le système
Menu principal → **6) Halt system**

### Reset aux paramètres d'usine
Menu principal → **4) Reset to factory defaults**

⚠️ **Attention** : Cette action efface toute la configuration !

---

## Connexion WebUI

Une fois pfSense configuré, accéder à l'interface web :
```
https://<IP_LAN>
```

Identifiants par défaut :
- **Username** : `admin`
- **Password** : `pfsense`

⚠️ **Changez le mot de passe immédiatement après la première connexion !**

---

## Sécurité

### Activer SSH sécurisé
Menu principal → **14) Enable Secure Shell (sshd)**

### Changer le mot de passe admin
Menu principal → **3) Reset admin account and password**

---

## Monitoring

### Voir les logs en temps réel
Menu principal → **10) Filter Logs**

### pfTop (monitoring ressources)
Menu principal → **9) pfTop**

### Ping un hôte
Menu principal → **7) Ping host**

---

## VPN OpenVPN

### Configuration Site-à-Site
À faire via WebUI : **VPN > OpenVPN > Wizards**

Paramètres recommandés :
- **Tunnel Network** : `10.0.0.0/30`
- **Encryption** : AES-256-GCM
- **Auth** : RSA-4096 + Certificats

---

## Dépannage

### Vérifier l'état du réseau
```sh
ifconfig
```

### Tester la connectivité
```sh
ping -c 4 8.8.8.8
```

### Voir les routes
```sh
netstat -rn
```

### Vérifier les services actifs
```sh
ps aux | grep pf
```

### Redémarrer les services réseau
```sh
/etc/rc.reload_all
```

---

## Notes

- Les modifications via shell ne sont pas toujours persistantes
- Privilégier la **WebUI** pour les configurations complexes
- Toujours sauvegarder la config avant modifications majeures (**Diagnostics > Backup & Restore**)
