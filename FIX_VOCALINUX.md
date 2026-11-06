# VocaLinux : CUDA, touche Copilot et accès aux paramètres

Ce document récapitule la configuration réalisée pour utiliser VocaLinux avec une carte NVIDIA, le lancer en arrière-plan et accéder à ses paramètres malgré un menu d’icône non fonctionnel sous KDE.

Les chemins personnels sont anonymisés. Dans les commandes shell, `~` et `$HOME` désignent le dossier personnel. Dans les fichiers `.desktop`, remplacer **`/home/UTILISATEUR` par le chemin absolu réel du dossier personnel** : le champ `Exec` ne développe pas automatiquement `~` ni `$HOME`.

## 1. Environnement et diagnostic initial

Configuration observée pendant l’installation :

| Élément | Version ou valeur |
| --- | --- |
| Système | Kali Linux, session KDE Plasma / Wayland |
| Carte graphique | NVIDIA GeForce RTX 4050 Laptop GPU, environ 6 Go de VRAM |
| Pilote NVIDIA | 550.163.01 |
| CUDA Toolkit (`nvcc`) | 12.4 |
| Python système | 3.14 |
| VocaLinux | Paquet système 0.18.5-1 |
| Moteur de transcription | `whisper_cpp` |
| Bibliothèque Python | `pywhispercpp` 1.5.0 |
| Langue et modèle | Français, `small` |

Le moteur fourni avec le paquet système était compilé uniquement pour le CPU. Son diagnostic indiquait :

```text
WHISPER : COREML = 0 | OPENVINO = 0 | CPU : OPENMP = 1 | REPACK = 1 |
```

Changer uniquement `whispercpp_gpu_device` dans le JSON ne pouvait pas ajouter un backend CUDA absent de la compilation.

**Solution retenue :** conserver le paquet système et compiler un moteur CUDA dans un environnement virtuel séparé. Aucun fichier du paquet sous `/usr/lib/vocalinux` ou `/usr/bin/vocalinux` n’a été modifié.

> La version CUDA affichée par `nvidia-smi` indique la compatibilité du pilote ; elle ne prouve pas à elle seule que le Toolkit est installé. Ici, sa présence a aussi été vérifiée avec `nvcc --version`.

## 2. Installation du moteur CUDA

### Prérequis

Ces outils étaient déjà disponibles sur la machine : Python avec `venv` et les en-têtes de développement, CMake, GCC/G++ 13, CUDA Toolkit 12.4 et les bibliothèques NVIDIA. Les dépendances de VocaLinux étaient fournies par le paquet système.

Pour vérifier les principaux éléments :

```bash
nvidia-smi
nvcc --version
python3 --version
cmake --version
g++-13 --version
```

### Créer l’environnement et compiler

```bash
python3 -m venv --system-site-packages ~/.config/vocalinux/venv-cuda

~/.config/vocalinux/venv-cuda/bin/python -m pip install --upgrade pip

CC=gcc-13 CXX=g++-13 CUDAHOSTCXX=g++-13 \
GGML_CUDA=1 CMAKE_ARGS="-DCMAKE_CUDA_ARCHITECTURES=89" \
~/.config/vocalinux/venv-cuda/bin/python -m pip install \
  --no-cache-dir --no-binary=pywhispercpp \
  "pywhispercpp==1.5.0" "pynput==1.8.2"
```

- `--system-site-packages` réutilise les dépendances Python installées par la distribution.
- `--no-binary=pywhispercpp` impose une compilation depuis les sources.
- `GGML_CUDA=1` active CUDA.
- L’architecture `89` correspond à la RTX 4050 (Ada, compute capability 8.9). Ne pas reprendre cette valeur sans vérification pour une autre carte.
- GCC/G++ 13 évitent les incompatibilités de CUDA 12.4 avec des compilateurs système trop récents.

Ces commandes téléchargent des paquets et compilent le moteur. Sur une autre machine, adapter les versions aux dépendances installées.

### Erreur rencontrée : `libwhisper.so.1` introuvable

Après compilation, un import direct échouait :

```text
ImportError: libwhisper.so.1: cannot open shared object file: No such file or directory
```

La bibliothèque était bien présente dans `venv-cuda/lib/python3.14/site-packages`, mais ce dossier n’était pas dans le chemin de recherche du chargeur de bibliothèques.

Vérification corrigée :

```bash
LD_LIBRARY_PATH="$HOME/.config/vocalinux/venv-cuda/lib/python3.14/site-packages" \
~/.config/vocalinux/venv-cuda/bin/python -c \
'import _pywhispercpp as w; print(w.whisper_print_system_info())'
```

Résultat obtenu, abrégé :

```text
ggml_cuda_init: found 1 CUDA devices
Device 0: NVIDIA GeForce RTX 4050 Laptop GPU, compute capability 8.9
WHISPER : ... CUDA : ARCHS = 890 ...
```

Le chargement CUDA et la détection de la carte ont donc été validés. Ce test seul ne constitue pas un test complet de transcription audio.

## 3. Lanceur CUDA permanent

Fichier : `~/.config/vocalinux/launch-cuda.sh`.

Contenu équivalent au script installé, avec le chemin personnel remplacé par `$HOME` :

```sh
#!/bin/sh
export LD_LIBRARY_PATH="$HOME/.config/vocalinux/venv-cuda/lib/python3.14/site-packages${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export PYTHONPATH=/usr/lib/vocalinux/app
export PYTHONNOUSERSITE=1
exec "$HOME/.config/vocalinux/venv-cuda/bin/python" -m vocalinux.main "$@"
```

Le rendre exécutable :

```bash
chmod +x ~/.config/vocalinux/launch-cuda.sh
```

Rôle des variables :

| Variable | Rôle |
| --- | --- |
| `LD_LIBRARY_PATH` | Rend accessibles les bibliothèques natives du moteur CUDA et conserve le chemin existant après ce dossier |
| `PYTHONPATH` | Charge le code du paquet VocaLinux installé dans `/usr/lib/vocalinux/app` |
| `PYTHONNOUSERSITE=1` | Évite les interférences des paquets Python du site utilisateur |

Le script utilise le Python de l’environnement CUDA et transmet les arguments, notamment `--start-minimized`.

**Il n’est pas nécessaire d’exporter ces variables à chaque lancement ni de les ajouter au profil du shell.** Elles sont définies uniquement pour VocaLinux et ses processus enfants.

> Le chemin `python3.14` est celui de cette installation. Après une mise à niveau majeure de Python, il peut être nécessaire de recréer l’environnement et d’adapter ce chemin. Les chemins `/usr/lib/vocalinux/app` sont également propres au paquet utilisé ici.

## 4. Menu d’applications et démarrage automatique

### Entrée principale : `vocalinux.desktop`

Copie de référence : `~/.config/vocalinux/vocalinux.desktop`.

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=Vocalinux
Comment=Voice dictation for Linux (CUDA)
GenericName=Voice Typing
Exec=/home/UTILISATEUR/.config/vocalinux/launch-cuda.sh
Icon=vocalinux
Terminal=false
Categories=Utility;Accessibility;AudioVideo;
Keywords=voice;typing;dictation;speech;recognition;transcription;accessibility;
StartupNotify=true
```

Après remplacement du chemin générique, installer l’entrée :

```bash
install -D -m 644 ~/.config/vocalinux/vocalinux.desktop \
  ~/.local/share/applications/vocalinux.desktop
```

Cette entrée utilisateur prend la priorité sur l’entrée système de même nom. Lancer **Vocalinux depuis le menu** utilise ainsi le moteur CUDA.

### Démarrage de session : `vocalinux-autostart.desktop`

Copie de référence : `~/.config/vocalinux/vocalinux-autostart.desktop`.

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=Vocalinux
Comment=Voice dictation for Linux (CUDA)
Exec=/home/UTILISATEUR/.config/vocalinux/launch-cuda.sh --start-minimized
Icon=vocalinux
Terminal=false
StartupNotify=false
X-GNOME-Autostart-enabled=true
```

Lors de la configuration, l’ancienne entrée de démarrage a été sauvegardée sous `~/.config/vocalinux/vocalinux-autostart.original.desktop` avant remplacement.

Pour reproduire l’installation, après adaptation du champ `Exec` :

```bash
mkdir -p ~/.config/autostart

# Si une ancienne entrée existe, la sauvegarder avant remplacement.
if [ -f ~/.config/autostart/vocalinux.desktop ]; then
  cp -n ~/.config/autostart/vocalinux.desktop \
    ~/.config/vocalinux/vocalinux-autostart.original.desktop
fi

install -m 644 ~/.config/vocalinux/vocalinux-autostart.desktop \
  ~/.config/autostart/vocalinux.desktop
```

Le fichier de sauvegarde correspond à l’ancienne configuration locale ; son contenu n’est pas nécessaire pour faire fonctionner CUDA.

> La commande système `/usr/bin/vocalinux` n’a pas été remplacée. L’appeler directement peut toujours utiliser le moteur CPU. Utiliser le menu ou `~/.config/vocalinux/launch-cuda.sh` pour le lancement CUDA. Vérifier également le fichier d’autostart si les paramètres de démarrage automatique sont modifiés ultérieurement dans l’application.

## 5. Configuration : français, arrière-plan et touche Copilot

Fichier : `~/.config/vocalinux/config.json`.

Les extraits ci-dessous présentent les réglages concernés ; **ne pas remplacer tout le fichier par ces seuls extraits**. Conserver les autres options existantes.

### Moteur et GPU

```json
{
    "speech_recognition": {
        "engine": "whisper_cpp",
        "language": "fr",
        "model_size": "small",
        "whisper_cpp_model_size": "small"
    },
    "advanced": {
        "whispercpp_gpu_device": null
    }
}
```

`null` conserve la sélection automatique du GPU. Le modèle `small` a été conservé pour commencer, sans augmenter inutilement les besoins en mémoire.

### Arrière-plan sans overlay

```json
{
    "ui": {
        "start_minimized": true,
        "show_notifications": true,
        "show_missing_tray_warning": true,
        "show_overlay": false
    },
    "general": {
        "autostart": true,
        "first_run": false
    }
}
```

- `show_overlay: false` masque l’indicateur flottant pendant la dictée.
- `start_minimized: true` demande un démarrage minimisé.
- Les notifications et l’icône de zone de notification ne sont pas désactivées.
- Le démarrage automatique est assuré par le fichier `.desktop` décrit plus haut.

Pour modifier le JSON manuellement, quitter d’abord l’application afin d’éviter qu’elle ne réenregistre une ancienne configuration, puis la relancer.

### Raccourci Copilot à la place d’AltGr

Le raccourci initial était `right_alt+right_alt`. Après enregistrement de la touche Copilot dans l’interface, le fichier contient :

```json
{
    "shortcuts": {
        "toggle_recognition": "shift+super+f23",
        "mode": "push_to_talk",
        "disable_internal_hotkey": false,
        "pass_shortcut_to_apps": false,
        "language_shortcuts": []
    }
}
```

Sur ce clavier, Copilot émet donc **Shift + Super + F23**. Cette combinaison n’est pas nécessairement identique sur tous les claviers.

Pour enregistrer une autre touche :

1. Ouvrir **Paramètres VocaLinux** depuis le menu d’applications.
2. Aller dans la section des raccourcis.
3. Dans **Shortcut Key**, choisir **Custom Shortcut**.
4. Cliquer sur **Record**, puis appuyer sur la touche souhaitée.
5. Valider avec **Set** si nécessaire.

Le mode **Push-to-Talk** permet de maintenir la touche pendant la parole, puis de la relâcher pour transcrire.

## 6. Accès aux paramètres sans utiliser l’icône KDE

### Problème rencontré

VocaLinux était bien actif, mais cliquer sur son icône ne permettait pas d’accéder au menu. L’inspection D-Bus a montré un menu exposé via `org.gtk.Menus` et des actions via `org.gtk.Actions`. Une incompatibilité de présentation de ce menu dans KDE a été suspectée ; le paquet système n’a pas été modifié pour la corriger.

Une commande D-Bus directe a permis d’activer l’action **Settings**. Le script ci-dessous automatise cette opération et recherche la connexion courante, au lieu de conserver un identifiant D-Bus temporaire qui changerait après un redémarrage.

### Script : `open-settings.py`

Fichier : `~/.config/vocalinux/open-settings.py`.

```python
#!/usr/bin/python3
"""Open the running VocaLinux settings through its exported tray action."""
import sys

from gi.repository import Gio, GLib


def main():
    bus = Gio.bus_get_sync(Gio.BusType.SESSION, None)

    def call(service, path, interface, method, parameters=None):
        return bus.call_sync(
            service, path, interface, method, parameters, None,
            Gio.DBusCallFlags.NONE, 3000, None,
        ).unpack()

    items = call(
        "org.kde.StatusNotifierWatcher", "/StatusNotifierWatcher",
        "org.freedesktop.DBus.Properties", "Get",
        GLib.Variant("(ss)", ("org.kde.StatusNotifierWatcher", "RegisteredStatusNotifierItems")),
    )[0]
    for item in items:
        service, separator, suffix = item.partition("/")
        path = "/" + suffix if separator else "/StatusNotifierItem"
        try:
            properties = call(
                service, path, "org.freedesktop.DBus.Properties", "GetAll",
                GLib.Variant("(s)", ("org.kde.StatusNotifierItem",)),
            )[0]
            if properties.get("Id") != "vocalinux":
                continue
            menu = properties["Menu"]
            layouts = call(
                service, menu, "org.gtk.Menus", "Start",
                GLib.Variant("(au)", ([0],)),
            )[0]
            try:
                for _group, _menu_id, entries in layouts:
                    for entry in entries:
                        if entry.get("label") == "Settings":
                            action = entry["action"].removeprefix("indicator.")
                            call(
                                service, menu, "org.gtk.Actions", "Activate",
                                GLib.Variant("(sava{sv})", (action, [], {})),
                            )
                            return
            finally:
                call(service, menu, "org.gtk.Menus", "End", GLib.Variant("(au)", ([0],)))
        except GLib.Error:
            continue
    raise RuntimeError("Lance VocaLinux avant d’ouvrir ses paramètres.")


if __name__ == "__main__":
    try:
        main()
    except (GLib.Error, RuntimeError) as error:
        message = str(error)
        print(message, file=sys.stderr)
        notification = Gio.Notification.new("Paramètres VocaLinux")
        notification.set_body(message)
        app = Gio.Application(application_id="com.vocalinux.SettingsLauncher", flags=Gio.ApplicationFlags.NON_UNIQUE)
        app.register(None)
        app.send_notification("settings-error", notification)
        Gio.DBusConnection.flush_sync(Gio.bus_get_sync(Gio.BusType.SESSION, None), None)
        sys.exit(1)
```

Ce script utilise le Python système et PyGObject (`gi`, fourni sur cette installation). Il n’a pas besoin de charger CUDA.

**Limites :** VocaLinux doit déjà tourner dans la même session utilisateur. Le script est adapté au menu GTK observé dans cette version ; une mise à jour changeant le backend, le nom de l’action ou son libellé peut nécessiter une adaptation.

### Entrée de menu : `vocalinux-settings.desktop`

Copie de référence : `~/.config/vocalinux/vocalinux-settings.desktop`.

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=Vocalinux Settings
Name[fr]=Paramètres VocaLinux
Comment=Open the settings of the running VocaLinux instance
Comment[fr]=Ouvrir les paramètres de VocaLinux en cours d’exécution
Exec=/usr/bin/python3 /home/UTILISATEUR/.config/vocalinux/open-settings.py
Icon=vocalinux
Terminal=false
Categories=Settings;
Keywords=vocalinux;settings;preferences;paramètres;raccourci;
StartupNotify=false
```

Après remplacement du chemin générique :

```bash
install -D -m 644 ~/.config/vocalinux/vocalinux-settings.desktop \
  ~/.local/share/applications/vocalinux-settings.desktop
```

### Utilisation quotidienne

1. Lancer **Vocalinux** depuis le menu, sauf s’il tourne déjà au démarrage de session.
2. Rechercher **Paramètres VocaLinux** dans le menu d’applications.
3. Cliquer sur cette entrée ; elle peut aussi être épinglée aux favoris.

Alternative en terminal :

```bash
/usr/bin/python3 ~/.config/vocalinux/open-settings.py
```

## 7. Vérification et dépannage

### Surveiller le GPU pendant une dictée

```bash
watch -n 1 nvidia-smi
```

Le processus Python devrait utiliser de la mémoire GPU lorsque le modèle est chargé. Avec le modèle `small`, les pics de calcul peuvent être brefs. Une activité CPU résiduelle est normale pour la capture audio et les autres tâches.

### Fermer une instance qui ne répond plus à l’interface

Le nom du processus peut être `python`, pas `vocalinux`. Identifier précisément l’instance :

```bash
pgrep -af 'python.*-m vocalinux[.]main'
```

Repérer son PID, puis remplacer `PID` dans les commandes suivantes par le nombre trouvé :

```bash
kill PID
```

Si elle ne s’arrête pas après quelques secondes, en dernier recours :

```bash
kill -9 PID
```

Ne pas utiliser `killall python` : cela arrêterait aussi d’autres applications Python. Ne pas réutiliser un ancien PID sans vérification, car il change et peut être réattribué.

Relancer ensuite depuis le menu d’applications pour conserver CUDA.

### Vérifier les fichiers

```bash
python3 -m json.tool ~/.config/vocalinux/config.json > /dev/null
sh -n ~/.config/vocalinux/launch-cuda.sh

desktop-file-validate ~/.local/share/applications/vocalinux.desktop
desktop-file-validate ~/.config/autostart/vocalinux.desktop
desktop-file-validate ~/.local/share/applications/vocalinux-settings.desktop
```

La validation de l’entrée principale peut produire une simple indication concernant plusieurs catégories principales. Cela n’empêche pas le lancement.

### Validations réalisées pendant la configuration

- Compilation et installation de `pywhispercpp` avec CUDA réussies.
- Chargement du moteur et détection de la RTX 4050 réussis après ajout du chemin des bibliothèques.
- Syntaxe du lanceur shell et des fichiers `.desktop` vérifiée.
- JSON validé après désactivation de l’overlay et activation du démarrage minimisé.
- Ouverture directe des paramètres par D-Bus testée ; accès confirmé par l’utilisateur.
- Lanceur permanent des paramètres testé avec succès.
- Enregistrement de la touche confirmé dans le JSON : `shift+super+f23`.

Le redémarrage complet de la session et une mesure de performances de transcription n’ont pas fait l’objet d’un test formel dans ces étapes.

## 8. Inventaire des fichiers

```text
~/.config/vocalinux/
├── README.md
├── config.json
├── launch-cuda.sh
├── open-settings.py
├── vocalinux.desktop
├── vocalinux-autostart.desktop
├── vocalinux-autostart.original.desktop
├── vocalinux-settings.desktop
└── venv-cuda/

~/.local/share/applications/
├── vocalinux.desktop
└── vocalinux-settings.desktop

~/.config/autostart/
└── vocalinux.desktop
```

Les fichiers `.desktop` du dossier de configuration servent de copies de référence. Le menu et le démarrage de session utilisent les copies installées dans les dossiers correspondants : après une modification, recopier le fichier concerné.

**En résumé :** le paquet système fournit l’application, l’environnement virtuel fournit le moteur CUDA, le lanceur configure automatiquement les chemins et le raccourci « Paramètres VocaLinux » contourne le menu de notification problématique. La touche Copilot déclenche la dictée en maintien, sans overlay.
