---

# Réparation de KDE Connect sur Kali Linux

Ce guide permet de résoudre l'erreur d'activation D-Bus (`org.kde.kdeconnect was not provided by any .service files`) et les problèmes de permissions Bluetooth sur Kali Linux.

## Prérequis
* Disposer des privilèges `sudo` sur la machine.
* Avoir installé `kdeconnect` via le gestionnaire de paquets APT.

## Procédure de Résolution

### 1. Restauration du service D-Bus
Le système ne parvient pas à localiser le démon KDE Connect. Il est nécessaire de recréer manuellement son fichier d'enregistrement D-Bus.

1. Créez et ouvrez le fichier de service :
   ```bash
   sudo nano /usr/share/dbus-1/services/org.kde.kdeconnect.service
   ```

2. Ajoutez la configuration suivante à l'intérieur :
   ```ini
   [D-BUS Service]
   Name=org.kde.kdeconnect
   Exec=/usr/bin/kdeconnectd
   ```
3. Sauvegardez et quittez (`Ctrl+O`, `Entrée`, puis `Ctrl+X`).

### 2. Attribution des permissions Bluetooth
Par mesure de sécurité, Kali Linux restreint les capacités réseau de certaines applications. Autorisez explicitement le démon à interagir avec la pile Bluetooth (correction de l'erreur `CAP_NET_ADMIN`) :

```bash
sudo setcap cap_net_admin,cap_net_raw+eip /usr/bin/kdeconnectd
```

### 3. Application des modifications et Redémarrage
Réinitialisez l'environnement pour forcer la prise en compte de la nouvelle configuration :

```bash
# Arrêter les instances existantes
pkill -f kdeconnectd

# Recharger la configuration D-Bus de l'utilisateur
systemctl --user reload-or-restart dbus

# Lancer l'application en arrière-plan
kdeconnect-app &
```

## Dépannage Réseau (Optionnel)
Si les appareils ne se détectent toujours pas malgré la résolution des erreurs de l'application, assurez-vous que le pare-feu de Kali Linux n'obstrue pas les flux réseau :

```bash
sudo ufw allow 1714:1764/udp
sudo ufw allow 1714:1764/tcp
sudo ufw reload
```
