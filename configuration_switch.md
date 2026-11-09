
---

# RAPPORT TECHNIQUE D’INTERVENTION
## Sécurisation des accès administratifs et audit des services réseau – Cisco Catalyst 2960

**Date de l’intervention :** 9 novembre 2026  
**Environnement d'administration :** Kali Linux  
**Équipements concernés :** 6 commutateurs Cisco Catalyst 2960  
**Nature de l'intervention :** Renforcement de la sécurité, activation de SSH, désactivation des services non sécurisés et vérification réseau.  
**Statut :** <badge color="success">Sécurisation réalisée et vérifiée sur les six équipements</badge>

---

## 1. Contexte et objectifs

À la suite de la précédente intervention ayant permis de rétablir le fonctionnement de l'infrastructure réseau, une opération complémentaire a été menée afin de sécuriser les accès administratifs des six commutateurs Cisco Catalyst 2960.

Les équipements utilisaient initialement Telnet pour l'administration à distance. Ce protocole ne chiffre ni les identifiants ni les communications, ce qui représente un risque pour la sécurité du réseau.

L'intervention avait pour objectifs :

- Remplacer l'administration Telnet par SSH version 2.
- Créer un compte administrateur local avec privilèges de niveau 15.
- Protéger le mode privilégié avec `enable secret`.
- Sécuriser l'accès physique à la console.
- Désactiver Telnet sur les lignes VTY.
- Désactiver les services HTTP et HTTPS.
- Générer des clés cryptographiques RSA de 2048 bits.
- Activer la protection des mots de passe simples dans la configuration.
- Définir une bannière d'avertissement MOTD.
- Limiter la durée d'inactivité des sessions administratives.
- Sauvegarder les configurations.
- Centraliser les paramètres de connexion SSH dans Kali Linux.
- Vérifier les services exposés sur les six équipements avec Nmap.

## 2. Inventaire des équipements concernés

L'intervention a concerné les six commutateurs suivants :

| Équipement | Adresse IP de gestion | Adresse MAC |
|---|---|---|
| SW211 | 192.168.1.211 | 00:14:A9:4B:C7:40 |
| SW212 | 192.168.1.212 | 00:1C:57:14:DC:40 |
| SW213 | 192.168.1.213 | 00:1A:A2:3B:08:40 |
| SW215 | 192.168.1.215 | 00:19:E7:77:AF:C0 |
| SW216 | 192.168.1.216 | 00:1C:0F:E4:74:41 |
| SW217 | 192.168.1.217 | 00:1C:0F:9A:8A:C0 |

Les six équipements sont administrés depuis le réseau local, à partir d'un poste Kali Linux.

La version IOS détaillée communiquée pour SW211 est :

- Modèle : Cisco WS-C2960-24TT-L.
- IOS : 15.0(2)SE11.
- Image : C2960-LANBASEK9-M.

Les commandes de configuration détaillées ci-dessous correspondent à la procédure appliquée aux six commutateurs.

## 3. Configuration de la sécurité administrative

### 3.1. Définition du domaine

Le domaine utilisé pour la génération des clés cryptographiques a été défini :

```text
ip domain-name mon-assistant-perso.local
```

Cette configuration permet notamment de générer les clés RSA nécessaires au fonctionnement de SSH.

### 3.2. Création du compte administrateur

Un compte administrateur local a été créé sur chaque commutateur :

```text
username it_map privilege 15 secret MOT_DE_PASSE_ADMIN
```

Paramètres :

| Paramètre | Valeur |
|---|---|
| Identifiant | it_map |
| Niveau de privilège | 15 |
| Authentification | Secret local |
| Mot de passe | Non communiqué dans le rapport |

Le niveau de privilège 15 permet d'accéder aux fonctions administratives complètes du commutateur.

### 3.3. Protection du mode privilégié

Un nouveau mot de passe a été défini pour le mode privilégié :

```text
enable secret NOUVEAU_MOT_DE_PASSE_PRIVILEGE
```

L'utilisation de `enable secret` permet de protéger le mode privilégié avec un secret haché.

### 3.4. Protection des mots de passe

La commande suivante a été appliquée :

```text
service password-encryption
```

Elle permet de masquer les mots de passe simples dans la configuration.

Il convient de préciser que le mécanisme Type 7 utilisé par cette commande est réversible et ne constitue pas un chiffrement fort. Les directives `secret` restent préférables pour les comptes et les accès privilégiés.

## 4. Activation et configuration de SSH

### 4.1. Génération des clés RSA

Sur chaque commutateur, une paire de clés RSA de 2048 bits a été générée :

```text
crypto key generate rsa modulus 2048
```

Ces clés permettent aux clients SSH de vérifier l'identité cryptographique des équipements.

### 4.2. Activation de SSH version 2

Les commandes suivantes ont été appliquées :

```text
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```

Paramètres retenus :

| Paramètre | Configuration |
|---|---|
| Protocole | SSH version 2 |
| Taille des clés RSA | 2048 bits |
| Délai d'authentification | 60 secondes |
| Tentatives d'authentification | 3 |

### 4.3. Configuration des lignes VTY

L'authentification locale a été activée sur les lignes VTY :

```text
line vty 0 15
login local
transport input all
exec-timeout 5 0
exit
```

Cette configuration initiale a permis de préparer SSH tout en conservant temporairement l'accès Telnet pendant les tests.

Après validation de SSH, la configuration définitive suivante a été appliquée :

```text
line vty 0 15
login local
transport input ssh
exec-timeout 5 0
exit
```

Ainsi, les lignes VTY n'acceptent plus les connexions Telnet.

Le délai d'inactivité des sessions administratives a été fixé à cinq minutes.

## 5. Désactivation des services HTTP et HTTPS

Les serveurs Web intégrés aux commutateurs ont été désactivés.

Commandes appliquées :

```text
no ip http server
no ip http secure-server
```

Cette mesure permet de supprimer les interfaces d'administration Web HTTP et HTTPS et de réduire les services exposés sur le réseau.

L'administration distante est désormais prévue exclusivement par SSH.

## 6. Sécurisation de l'accès console

L'accès physique à la console a également été protégé.

Configuration appliquée :

```text
line console 0
password NOUVEAU_MOT_DE_PASSE_CONSOLE
login
exec-timeout 5 0
logging synchronous
exit
```

Mesures mises en place :

- Définition d'un nouveau mot de passe console.
- Authentification obligatoire.
- Déconnexion automatique après cinq minutes d'inactivité.
- Amélioration de la lisibilité des messages système grâce à `logging synchronous`.

La commande `service password-encryption` masque le mot de passe console dans la configuration, avec les limites du chiffrement réversible Type 7.

## 7. Configuration de la bannière MOTD

Une bannière de sécurité a été configurée sur les équipements :

```text
banner motd *ACCES RESTREINT - RESEAU : Equipement reserve au personnel autorise.*
```

Cette bannière informe les utilisateurs que les commutateurs sont réservés au personnel autorisé.

Elle constitue un avertissement affiché lors des connexions administratives.

## 8. Administration centralisée depuis Kali Linux

Afin de simplifier les connexions aux six équipements, un fichier de configuration SSH personnalisé a été créé :

```bash
nano ~/.ssh/config
```

Les paramètres communs ont été définis :

```ssh-config
# Paramètres communs aux commutateurs Cisco Catalyst 2960
Host sw211 sw212 sw213 sw215 sw216 sw217
    User it_map
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc,aes256-cbc
```

Les correspondances entre les alias et les adresses IP ont été ajoutées :

```ssh-config
Host sw211
    HostName 192.168.1.211

Host sw212
    HostName 192.168.1.212

Host sw213
    HostName 192.168.1.213

Host sw215
    HostName 192.168.1.215

Host sw216
    HostName 192.168.1.216

Host sw217
    HostName 192.168.1.217
```

Les connexions peuvent désormais être effectuées directement avec les commandes suivantes :

```bash
ssh sw211
ssh sw212
ssh sw213
ssh sw215
ssh sw216
ssh sw217
```

Les paramètres cryptographiques sont centralisés dans le fichier de configuration local.

Les algorithmes utilisés sont anciens et nécessaires à la compatibilité avec les versions IOS des commutateurs concernés. Leur utilisation est limitée aux hôtes Cisco déclarés dans le fichier SSH.

### 8.1. Authentification par clé publique

L'utilisation de la clé personnelle Ed25519 de l'administrateur a été étudiée.

Cependant, la version IOS 15.0(2)SE11 des équipements concernés ne documente pas l'authentification utilisateur SSH par clé publique Ed25519.

L'authentification repose donc sur les comptes administrateurs locaux et leurs mots de passe.

## 9. Sauvegarde des configurations

Après les opérations de configuration, les paramètres ont été enregistrés dans la mémoire permanente des commutateurs avec :

```text
copy running-config startup-config
```

La sauvegarde de SW211 a notamment été confirmée par le retour :

```text
Building configuration...
[OK]
```

Les configurations ont été enregistrées sur les équipements concernés afin de conserver les paramètres après redémarrage.

## 10. Vérification de la sécurité avec Nmap

Une vérification réseau a été réalisée depuis Kali Linux le 9 novembre 2026 à 13 h 52–13 h 53 UTC.

L'objectif était de contrôler l'accessibilité des quatre principaux services d'administration :

- TCP/22 : SSH.
- TCP/23 : Telnet.
- TCP/80 : HTTP.
- TCP/443 : HTTPS.

Commande utilisée pour chaque équipement :

```bash
sudo nmap -Pn -p 22,23,80,443 ADRESSE_IP
```

### 10.1. Résultats détaillés

Les six scans ont donné les résultats suivants :

| Équipement | SSH 22 | Telnet 23 | HTTP 80 | HTTPS 443 |
|---|---|---|---|---|
| SW211 | OPEN | CLOSED | CLOSED | CLOSED |
| SW212 | OPEN | CLOSED | CLOSED | CLOSED |
| SW213 | OPEN | CLOSED | CLOSED | CLOSED |
| SW215 | OPEN | CLOSED | CLOSED | CLOSED |
| SW216 | OPEN | CLOSED | CLOSED | CLOSED |
| SW217 | OPEN | CLOSED | CLOSED | CLOSED |

### 10.2. Résultats des adresses MAC

Nmap a également permis de confirmer l'identification des six équipements comme appartenant au constructeur Cisco Systems.

| Équipement | Adresse MAC détectée |
|---|---|
| SW211 | 00:14:A9:4B:C7:40 |
| SW212 | 00:1C:57:14:DC:40 |
| SW213 | 00:1A:A2:3B:08:40 |
| SW215 | 00:19:E7:77:AF:C0 |
| SW216 | 00:1C:0F:E4:74:41 |
| SW217 | 00:1C:0F:9A:8A:C0 |

### 10.3. Analyse des résultats

Les résultats démontrent une homogénéité des services exposés sur les six équipements.

**SSH (port 22)**

Le port 22 est ouvert sur les six commutateurs. Le service SSH est donc accessible au niveau réseau sur chacun des équipements.

**Telnet (port 23)**

Le port 23 est fermé sur les six commutateurs. Aucune nouvelle connexion Telnet n'est acceptée.

**HTTP (port 80)**

Le port 80 est fermé sur les six commutateurs. Le serveur HTTP est désactivé.

**HTTPS (port 443)**

Le port 443 est fermé sur les six commutateurs. Le serveur HTTPS est désactivé.

### 10.4. Conclusion de l'audit Nmap

Le contrôle des quatre services réseau a donné le même résultat pour les six commutateurs.

Sur les 24 vérifications de ports effectuées :

- 6 ports SSH ouverts.
- 18 ports non accessibles (Telnet, HTTP et HTTPS fermés).

Le résultat correspond à la politique de sécurité recherchée : conserver SSH comme unique service d'administration parmi les quatre services testés.

Il est important de préciser que le scan Nmap confirme l'accessibilité réseau des services, mais ne vérifie pas à lui seul la réussite de l'authentification ni la validité de tous les paramètres internes.

## 11. Bilan général de l'intervention

La sécurisation a été réalisée sur les six équipements.

| Opération | SW211 | SW212 | SW213 | SW215 | SW216 | SW217 |
|---|---|---|---|---|---|---|
| Compte administrateur local | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| Privilège 15 | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| Enable secret | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| Clés RSA 2048 bits | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| SSH version 2 | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| Authentification locale VTY | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| Désactivation Telnet | Vérifié | Vérifié | Vérifié | Vérifié | Vérifié | Vérifié |
| Désactivation HTTP/HTTPS | Vérifié | Vérifié | Vérifié | Vérifié | Vérifié | Vérifié |
| Protection console | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |
| Bannière MOTD | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé | Réalisé |

La colonne « Vérifié » correspond aux résultats des scans Nmap. Les autres opérations correspondent à la procédure de configuration appliquée aux six équipements.

## 12. Recommandations complémentaires

Malgré les résultats positifs, plusieurs améliorations peuvent encore être envisagées.

### 12.1. Renforcement cryptographique

Les algorithmes SHA-1, RSA avec signature SSH ancienne et AES-CBC sont obsolètes ou moins robustes que les alternatives modernes.

Il est recommandé d'étudier une évolution des équipements ou de leur logiciel afin de pouvoir utiliser des algorithmes cryptographiques contemporains.

### 12.2. Authentification

Les mots de passe administratifs doivent être :

- Forts et uniques.
- Conservés dans un gestionnaire de mots de passe.
- Non partagés entre plusieurs utilisateurs.
- Exclus des fichiers de configuration accessibles aux utilisateurs non autorisés.

### 12.3. Sauvegardes

Il est recommandé de conserver une copie externe de la configuration de chaque commutateur, en plus de la sauvegarde dans la mémoire permanente.

### 12.4. Surveillance et journalisation

La mise en place d'une centralisation des journaux système permettrait de surveiller les connexions administratives, les erreurs des interfaces et les événements de sécurité.

### 12.5. Contrôle physique et réseau

Une vérification régulière de l'état des ports, des VLAN, des liaisons montantes et des éventuelles boucles réseau est recommandée.

---

## 13. Conclusion générale

L'intervention du 9 novembre 2026 a permis de sécuriser les accès administratifs des six commutateurs Cisco Catalyst 2960 de l'infrastructure.

Les équipements ont été configurés selon une politique de sécurité commune reposant sur SSH version 2, des comptes administrateurs locaux privilégiés, la protection des accès console et du mode privilégié, ainsi que la désactivation des services Telnet, HTTP et HTTPS.

L'administration a également été simplifiée grâce à la création d'un fichier SSH centralisé sur Kali Linux, permettant de se connecter aux six équipements au moyen de simples alias.

Enfin, un audit réseau effectué avec Nmap a confirmé que les six commutateurs exposent SSH sur le port 22 et que les ports Telnet, HTTP et HTTPS sont fermés.

**BILAN FINAL : LES SIX COMMUTATEURS ONT ÉTÉ CONFIGURÉS SELON LA POLITIQUE DE SÉCURITÉ DÉFINIE ET LES RÉSULTATS NMAP CONFIRMENT L'EXPOSITION EXCLUSIVE DE SSH PARMI LES QUATRE SERVICES CONTRÔLÉS.**

*Rapport technique établi le 9 novembre 2026.*