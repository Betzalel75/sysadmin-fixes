# RAPPORT TECHNIQUE D’INTERVENTION

## Diagnostic et rétablissement du réseau informatique Commutateurs Cisco Catalyst 2960

**Date de l’intervention :** 2 novembre 2026
**Nature de l’intervention :** Dépannage réseau, réinitialisation et reconfiguration de commutateurs
**Équipements concernés :** 3 commutateurs Cisco Catalyst 2960
**Environnement de travail :** Kali Linux
**État final :** Service réseau rétabli

---

## 1. Contexte de l’intervention

Une panne matérielle est survenue sur le commutateur principal de l’infrastructure réseau. Cet équipement, qui recevait directement l’arrivée Internet et assurait la connexion vers les autres commutateurs, ne présentait plus aucun signe d’alimentation ou de fonctionnement.

À la suite de cette panne, une première tentative de rétablissement a consisté à déplacer le câble d’arrivée Internet vers un autre commutateur Cisco Catalyst 2960.

Cependant, cette modification n’a pas permis de rétablir le fonctionnement du réseau. Les postes connectés aux ports FastEthernet ne parvenaient pas à communiquer avec la passerelle Internet ni à obtenir normalement leurs paramètres réseau par DHCP.

Un problème supplémentaire a été identifié : les commutateurs disposaient d’une configuration héritée de l’installation précédente, protégée par des identifiants d’administration qui n’avaient pas été transmis. L’ancien technicien chargé de l’installation n’était plus disponible.

Une intervention de diagnostic, de réinitialisation et de reconfiguration a donc été entreprise sur les trois commutateurs encore opérationnels.

## 2. Objectifs de l’intervention

- Identifier la cause de l’absence de connectivité sur les ports FastEthernet.
- Vérifier le fonctionnement des liaisons Ethernet et des commutateurs.
- Rechercher les possibilités d’accès à l’administration sans câble console.
- Supprimer les anciennes configurations devenues inaccessibles.
- Reconfigurer les trois équipements avec de nouvelles adresses IP de gestion et de nouveaux mots de passe.
- Restaurer l’accès au réseau local et à Internet.
- Vérifier l’accès à distance aux commutateurs.

## 3. Diagnostic initial

### 3.1. Vérification des connexions physiques

Les tests initiaux ont été réalisés avec le câblage suivant :

- Port GigabitEthernet 0/1 : connexion à la box Internet.
- Port GigabitEthernet 0/2 : connexion au poste Kali Linux.

Avec cette configuration, le poste pouvait obtenir une connectivité réseau et accéder à Internet.

En revanche, lorsqu’il était connecté à un port FastEthernet, la communication avec la passerelle ne fonctionnait plus.

### 3.2. Vérification de l’interface Ethernet sous Kali Linux

Les commandes suivantes ont été exécutées :

```bash
ip -4 addr
ip route
ip -br link show eth0
sudo ethtool eth0
```

Les résultats ont montré :

| Paramètre                | Résultat        |
| ------------------------ | --------------- |
| Interface réseau         | eth0            |
| État de l’interface      | UP, LOWER_UP    |
| Adresse IPv4 initiale    | 192.168.1.60/24 |
| Passerelle               | 192.168.1.1     |
| Vitesse sur FastEthernet | 100 Mb/s        |
| Mode duplex              | Full Duplex     |
| Auto-négociation         | Activée         |
| Détection de liaison     | Yes             |

Ces résultats ont confirmé que la liaison physique Ethernet fonctionnait sur le port FastEthernet testé.

Le problème ne provenait donc pas d’une absence totale de liaison physique sur ce port.

### 3.3. Tests de communication réseau

Les commandes suivantes ont été utilisées :

```bash
ping -I eth0 -c 4 192.168.1.1
ip neigh show dev eth0
```

Résultats :

```text
4 packets transmitted, 0 received
100% packet loss

192.168.1.1 FAILED
```

L’échec de la résolution ARP indiquait que le poste ne parvenait pas à joindre la passerelle sur son interface Ethernet.

Une interaction entre les interfaces Wi-Fi et Ethernet a également été relevée : un premier test sélectionnait l’adresse source du Wi-Fi. Cette observation a nécessité de distinguer les deux interfaces durant les investigations.

L’absence de réponse ARP sur FastEthernet orientait le diagnostic vers un problème de transmission de niveau 2, notamment une configuration VLAN inadaptée, un filtrage ou un autre paramètre du commutateur.

## 4. Recherche de l’administration des commutateurs

### 4.1. Problème d’accès console

L’accès console classique ne pouvait pas être utilisé, faute de câble console série adapté.

Un câble Ethernet RJ45 standard ne permet pas d’accéder directement au port CONSOLE du Cisco Catalyst 2960, celui-ci utilisant une liaison série et non Ethernet.

Il a donc été décidé de rechercher un accès distant par le réseau local.

### 4.2. Analyse du réseau avec Nmap

Un premier scan a été effectué :

```bash
sudo nmap -sn -e eth0 192.168.1.0/24
```

Plusieurs équipements du réseau ont été identifiés, mais aucun ne correspondait clairement au Cisco recherché.

Un second scan a permis de rechercher les services d’administration :

```bash
sudo nmap -sT -Pn --open -p 22,23,80,443 192.168.1.0/24
```

L’adresse de gestion du Cisco n’a pas été identifiée dans ce sous-réseau.

### 4.3. Découverte grâce au protocole CDP

La commande suivante a été exécutée :

```bash
sudo timeout 120 tcpdump -ni eth0 -e -vvv \
'ether dst 01:00:0c:cc:cc:cc'
```

L’analyse des annonces Cisco Discovery Protocol (CDP) a permis d’identifier précisément un commutateur.

**Informations découvertes :**

| Paramètre             | Valeur                    |
| --------------------- | ------------------------- |
| Modèle                | Cisco WS-C2960-24TT-L     |
| Nom d’hôte            | meeting-215.meeting-group |
| IOS                   | 15.0(2)SE11               |
| Adresse IP de gestion | 10.71.1.215               |
| Domaine VTP           | Meeting                   |
| VLAN natif annoncé    | 1                         |
| Port connecté         | GigabitEthernet0/2        |

Cette découverte a confirmé la présence d’une ancienne configuration réseau, avec une adresse de gestion distincte du réseau actuel `192.168.1.0/24`.

## 5. Accès à l’ancienne adresse de gestion

Une adresse IP secondaire a été temporairement attribuée au poste Kali Linux :

```bash
sudo ip addr add 10.71.1.100/24 dev eth0
```

La communication avec le switch a ensuite été vérifiée :

```bash
ping -I 10.71.1.100 -c 4 10.71.1.215
```

**Résultat :**

```text
4 packets transmitted
4 received
0% packet loss
```

La commande :

```bash
ip neigh show 10.71.1.215 dev eth0
```

a confirmé la résolution ARP vers l’adresse MAC Cisco `00:19:E7:77:AF:C0`.

Ces tests ont établi que le commutateur était accessible depuis le réseau local, malgré sa configuration d’adressage différente.

### 5.1. Vérification des services d’administration

La commande suivante a été utilisée :

```bash
sudo nmap -sV -Pn -p 22,23,80,443 10.71.1.215
```

Résultats :

| Port | Service | État   |
| ---- | ------- | ------ |
| 22   | SSH     | Ouvert |
| 23   | Telnet  | Fermé  |
| 80   | HTTP    | Fermé  |
| 443  | HTTPS   | Fermé  |

L’ancien équipement était donc administrable par SSH, mais les identifiants de l’ancien administrateur n’étaient pas disponibles.

L’accès à la configuration existante n’a pas pu être obtenu.

## 6. Réinitialisation et reconfiguration

Compte tenu de l’absence des anciens identifiants, une remise à zéro des commutateurs a été retenue.

### 6.1. Utilisation du bouton MODE

Les opérations de réinitialisation et de préparation à la configuration ont été réalisées à l’aide du bouton physique MODE.

Au cours des manipulations, les voyants SYST et STAT ont été observés.

Le voyant SYST avait notamment présenté un état orange persistant avant de revenir à un fonctionnement vert après redémarrage électrique.

L’activation d’Express Setup a été confirmée lorsque les voyants STAT, DUPLX et SPEED sont devenus verts fixes.

### 6.2. Accès à Express Setup

La configuration initiale a été réalisée depuis le navigateur du poste Kali Linux, au moyen d’un câble Ethernet RJ45 classique.

L’interface Express Setup a permis de définir les nouveaux paramètres d’administration des commutateurs.

### 6.3. Paramètres configurés

Pour chacun des trois commutateurs :

- Attribution d’un nom d’équipement.
- Configuration d’une nouvelle adresse IP de gestion sur le VLAN 1.
- Définition de nouveaux mots de passe d’administration.
- Activation de l’accès distant Telnet.
- Remise en service des connexions Ethernet.

**Nouvelle configuration d’administration :**

| Commutateur | Nom configuré | Adresse IP de gestion | Accès vérifié |
| ----------- | ------------- | --------------------- | ------------- |
| Switch 1    | SW211         | 192.168.1.211         | Telnet        |
| Switch 2    | SW213         | 192.168.1.213         | Telnet        |
| Switch 3    | sw215         | 192.168.1.215         | Telnet        |

Les mots de passe définis ne sont volontairement pas inclus dans ce rapport pour des raisons de sécurité.

## 7. Vérification de l’accès aux trois commutateurs

Après configuration, des connexions Telnet ont été effectuées depuis Kali Linux.

### 7.1. Commutateur SW211

```text
telnet 192.168.1.211
Connected to 192.168.1.211.

User Access Verification
Password:
SW211>
```

**Résultat :** accès au mode utilisateur Cisco obtenu.

### 7.2. Commutateur SW213

```text
telnet 192.168.1.213
Connected to 192.168.1.213.

User Access Verification
Password:
SW213>
```

**Résultat :** accès au mode utilisateur Cisco obtenu.

### 7.3. Commutateur sw215

```text
telnet 192.168.1.215
Connected to 192.168.1.215.

User Access Verification
Password:
sw215>
```

**Résultat :** accès au mode utilisateur Cisco obtenu.

Ces tests ont permis de confirmer que les trois commutateurs disposaient désormais d’adresses de gestion fonctionnelles et que leurs accès Telnet étaient opérationnels.

## 8. Résultat final de l’intervention

Après la reconfiguration des trois commutateurs et le rebranchement des câbles réseau ainsi que de l’arrivée Internet, le fonctionnement du réseau a été rétabli.

**Résultats constatés :**

- Rétablissement de la connectivité réseau.
- Fonctionnement retrouvé des ports FastEthernet et Gigabit utilisés.
- Rétablissement de l’accès à Internet.
- Fonctionnement de l’attribution des adresses IP sur le réseau.
- Nouvelle configuration de gestion des trois commutateurs.
- Nouveaux identifiants d’administration définis.
- Accès Telnet confirmé sur les trois équipements.

L’objectif principal de l’intervention a donc été atteint.

## 9. Analyse de la cause probable

La panne initiale était une défaillance matérielle du commutateur principal, qui ne s’allumait plus.

Après le déplacement de l’arrivée Internet vers un autre commutateur, la connectivité n’avait pas été restaurée.

L’analyse a révélé l’existence d’une configuration héritée, incluant notamment un domaine VTP nommé `Meeting` et une ancienne adresse de gestion `10.71.1.215`.

Il est probable que certains paramètres de cette installation, notamment les affectations VLAN ou les modes des ports, étaient incompatibles avec le nouveau branchement de l’arrivée Internet.

La reconfiguration des commutateurs a permis de rétablir les communications.

Cependant, **la cause logique exacte de l’absence d’accès Internet n’a pas pu être confirmée**, car la configuration initiale complète n’a pas été récupérée avant la réinitialisation.

## 10. Recommandations techniques

### 10.1. Sécurisation des accès

L’administration Telnet fonctionne, mais ce protocole transmet les identifiants et les données sans chiffrement.

Il est recommandé de configurer SSH sur les trois commutateurs et de désactiver ensuite Telnet, après validation des accès SSH.

### 10.2. Sauvegarde des configurations

Les configurations doivent être enregistrées dans la mémoire permanente des équipements :

```text
enable
copy running-config startup-config
```

Il est également recommandé d’exporter les configurations dans un emplacement sécurisé.

### 10.3. Documentation de l’infrastructure

Réaliser un schéma de l’installation précisant :

- L’emplacement de chaque commutateur.
- Les connexions entre équipements.
- Les ports Gigabit utilisés pour les liaisons montantes.
- Les adresses IP de gestion.
- Les VLAN configurés.
- Les connexions vers les postes et autres matériels.

### 10.4. Vérification de la segmentation réseau

Les paramètres VLAN doivent être contrôlés afin de s’assurer que la remise en service n’a pas supprimé une séparation nécessaire entre différents équipements ou services.

### 10.5. Contrôle des équipements

Il est conseillé de vérifier les journaux système, l’état des interfaces, les erreurs de ports et les éventuelles boucles réseau.

Le commutateur principal hors service devra faire l’objet d’un diagnostic matériel séparé ou être remplacé.

---

## 11. Conclusion

L’intervention a permis de diagnostiquer et de résoudre un incident réseau survenu à la suite de la panne matérielle du commutateur principal.

La recherche de l’ancienne configuration, notamment grâce aux outils Nmap et tcpdump sous Kali Linux et au protocole Cisco CDP, a permis d’identifier le matériel, son adresse de gestion et les services d’administration disponibles.

En l’absence des anciens identifiants et d’un câble console, l’utilisation du bouton MODE et d’Express Setup a constitué une solution adaptée à la remise en service.

Les trois commutateurs Cisco Catalyst 2960 ont ensuite été reconfigurés, avec de nouvelles adresses IP de gestion, de nouveaux mots de passe et un accès distant fonctionnel.

**L’infrastructure réseau est de nouveau opérationnelle, avec un accès Internet rétabli et une administration des commutateurs désormais disponible.**

**Statut final : INTERVENTION RÉUSSIE SERVICE RÉTABLI**
