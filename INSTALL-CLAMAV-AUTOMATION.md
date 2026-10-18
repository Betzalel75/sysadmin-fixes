---

# 🛡️ Automatisation de l'Antivirus ClamAV avec ClamUI (Flatpak) et Incron

Ce guide détaille la procédure complète pour mettre en place une solution antivirus performante, graphique et automatisée sous **Debian, Ubuntu, Linux Mint et dérivés**. 

L'objectif de cette configuration est d'avoir un **scan automatique en temps réel lors des téléchargements et des branchements de clés USB**, tout en conservant **0 Mo de RAM consommés au repos** (contrairement au démon On-Access de ClamAV qui mobilise en continu 1,5 Go de mémoire vive).

---

## 🏗️ Architecture de la Solution

1. **Moteur (`ClamAV`)** : Assure l'analyse des fichiers et gère la base de signatures virales.
2. **Interface Graphique (`ClamUI` Flatpak)** : Permet de piloter l'application visuellement et d'exécuter des analyses manuelles à la demande.
3. **Pont Système (`ClamUI Privileged Helper` natif)** : Permet à l'application isolée en Flatpak d'interagir avec les fichiers de configuration de la machine hôte.
4. **Planificateur d'Événements (`incron`)** : Surveille le système à l'aide d'événements du noyau Linux (`inotify`) et réveille ClamAV à la milliseconde près uniquement quand un fichier apparaît.

---

## 🛠️ Étape 1 : Installation du Moteur Antivirus Local

Commencez par installer le moteur de détection ClamAV de manière native sur votre système :

```bash
sudo apt update && sudo apt install clamav clamav-daemon clamav-freshclam -y
```

---

## 📦 Étape 2 : Installation de l'Interface Graphique ClamUI

Installez l'application graphique via le protocole Flatpak :

```bash
flatpak install flathub com.github.linx_systems.clamui -y
```

---

## 🔑 Étape 3 : Installation du Privileged Helper Natif

Pour résoudre les blocages de droits liés au confinement (sandbox) du Flatpak et lier l'interface à la configuration globale, installez manuellement le paquet compagnon `.deb`.

1. Téléchargez la version correspondante à votre application (exemple ici avec la v0.4.1) :
   ```bash
   wget https://github.com/linx-systems/clamui/releases/download/v0.4.1/clamui-privileged-helper_0.4.1_all.deb
   ```
2. Installez le paquet localement avec `apt` pour configurer le service de privilèges :
   ```bash
   sudo apt install ./clamui-privileged-helper_0.4.1_all.deb -y
   ```

---

## 🎯 Étape 4 : Configuration des Autorisations (Flatseal)

Pour s'assurer que ClamUI ait accès au système de fichiers de votre machine pour ses rapports :
1. Lancez **Flatseal** (installez-le via `flatpak install flathub com.github.tchx84.Flatseal` si nécessaire).
2. Sélectionnez **ClamUI**.
3. Dans la section **Filesystem**, activez les options suivantes :
   *   `All system files (host)`
   *   `All system configuration files (host-etc)`

---

## 🤖 Étape 5 : Automatisation des Scans sans impact RAM via `incron`

Pour éviter que l'antivirus ne tourne en arrière-plan et ne s'exécute qu'au moment précis de l'arrivée d'un fichier, nous utilisons le démon d'événements `incron`.

### 1. Installation de l'outil système
```bash
sudo apt install incron -y
```

### 2. Autorisation de l'utilisateur standard
Ajoutez votre nom d'utilisateur dans la liste des comptes autorisés à manipuler le démon :
```bash
echo "$USER" | sudo tee -a /etc/incron.allow
```

### 3. Création du dossier de quarantaine
Définissez l'emplacement où ClamAV déplacera automatiquement les fichiers suspects :
```bash
mkdir -p /tmp/quarantaine
```

### 4. Configuration des règles de surveillance
Ouvrez l'éditeur de table `incron` :
```bash
incrontab -e
```
*Choisissez votre éditeur de texte si demandé (ex : `nano`).*

Ajoutez les deux lignes suivantes tout en bas du fichier *(remplacez `betzalel` par votre nom d'utilisateur exact)* :
```text
/home/betzalel/Téléchargements IN_CLOSE_WRITE clamscan --infected --move=/tmp/quarantaine $@/$#
/media/betzalel IN_CREATE clamscan -r --infected --move=/tmp/quarantaine $@/$#
```

*   **Ligne 1 (Téléchargements)** : Scanne immédiatement n'importe quel fichier provenant d'un navigateur internet ou d'un terminal dès que son écriture est terminée (`IN_CLOSE_WRITE`).
*   **Ligne 2 (Périphériques USB)** : Déclenche un scan complet et récursif (`-r`) de n'importe quel support externe ou clé USB dès que son dossier de montage est créé par Linux (`IN_CREATE`).

Sauvegardez et quittez l'éditeur (`Ctrl+O`, `Entrée`, puis `Ctrl+X` sous Nano). Le système confirmera par : `incrontab: table updated`.

---

## 🧪 Étape 6 : Validation du bon fonctionnement

Pour vous assurer que le système intercepte bien les menaces, vous pouvez simuler la création d'un virus de test standard (EICAR) inoffensif directement dans votre dossier surveillé :

```bash
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > ~/Téléchargements/test_antivirus.txt
```

**Résultat attendu** : Moins d'une seconde après l'exécution de la commande, le fichier `test_antivirus.txt` doit **disparître** de votre dossier `Téléchargements` et se retrouver isolé dans `/tmp/quarantaine/`.
