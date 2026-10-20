# Configurer Codewhale avec Ollama en local — Guide débutant

Ce guide t'explique, **étape par étape et sans jargon**, comment faire tourner
Codewhale (l'agent IA de ton terminal) avec **Ollama** (un serveur qui fait
tourner les modèles d'IA **directement sur ta machine**, sans cloud).

À la fin, tu sauras aussi **changer de modèle** en une seule commande.

---

## 0. C'est quoi, en 30 secondes ?

- **Ollama** = un programme qui télécharge et fait tourner des modèles d'IA
  (comme un « moteur »). Il écoute sur `http://localhost:11434`.
- **Codewhale** = ton assistant IA de terminal. Par défaut il peut utiliser des
  modèles dans le cloud (DeepSeek, OpenAI…). On va lui dire : « utilise plutôt
  mon Ollama local ».
- **Un modèle** = le « cerveau » que Codewhale utilise. On en a plusieurs, et on
  peut basculer de l'un à l'autre.

> 💡 Avantage du local : **gratuit**, **privé** (tes données ne quittent pas ta
> machine), et ça marche **sans Internet**.

---

## 1. Ce qu'il faut avoir installé

Deux logiciels :

| Logiciel  | Rôle                                | Comment vérifier            |
| --------- | ----------------------------------- | --------------------------- |
| Ollama    | Serveur local de modèles IA         | `ollama --version`          |
| Codewhale | L'assistant IA de ton terminal      | `codewhale --version`       |

Ouvre un terminal et tape :

```bash
ollama --version
codewhale --version
```

- Si **Ollama** n'est pas installé → voir l'étape 2.
- Si **Codewhale** n'est pas installé → voir l'étape 3.

---

## 2. Installer Ollama

Selon ton système :

```bash
# Linux / WSL
curl -fsSL https://ollama.com/install.sh | sh
```

Pour **macOS** ou **Windows**, télécharge l'installateur depuis le site officiel :
👉 https://ollama.com/download

Vérifie que c'est bon :

```bash
ollama --version
```

---

## 3. Installer Codewhale

Codewhale s'installe via Cargo (l'outil de Rust) ou par un binaire. La façon la
plus courante :

```bash
cargo install codewhale
```

> Si `cargo` n'existe pas, installe d'abord Rust :
> `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
> puis **ferme et rouvre** ton terminal.

Vérifie que c'est bon :

```bash
codewhale --version
```

---

## 4. Lancer le serveur Ollama

Le serveur doit **tourner en permanence** pendant que tu utilises Codewhale.

Dans un terminal, lance :

```bash
ollama serve
```

> 💡 Laisse ce terminal **ouvert**. Ouvre un **deuxième terminal** pour la suite.

Vérifie qu'il répond :

```bash
curl http://localhost:11434/api/tags
```

Si tu vois une réponse en texte (même vide `{}`), c'est que le serveur tourne. ✅

---

## 5. Télécharger un (ou plusieurs) modèles

Tape `ollama pull` suivi du nom du modèle. Exemples :

```bash
# Modèle orienté code (recommandé pour commencer)
ollama pull qwen2.5-coder:7b

# Modèle de raisonnement (réfléchit "à voix haute")
ollama pull deepseek-r1:8b
```

> ⚠️ Chaque modèle fait plusieurs **Go** (4 à 5 Go ici). Le premier
> téléchargement peut prendre du temps selon ta connexion.

Pour voir **tous** tes modèles téléchargés :

```bash
ollama list
```

---

## 6. Pointer Codewhale vers Ollama

C'est le cœur de la configuration. Deux réglages à faire :

### 6.1 — Dire à Codewhale d'utiliser le serveur Ollama

```bash
codewhale config set provider ollama
```

### 6.2 — Dire à Codewhale où se trouve le serveur

```bash
codewhale config set providers.ollama.base_url http://localhost:11434
```

> 💡 Normalement cette adresse est déjà bonne, mais cette commande garantit
> qu'elle est définie.

### 6.3 — Vérifier la configuration

```bash
codewhale config list
```

Tu dois voir apparaître :

```text
provider = ollama
providers.ollama.base_url = http://localhost:11434
```

---

## 7. Choisir le modèle

Maintenant on dit à Codewhale **quel modèle** utiliser :

```bash
codewhale config set default_text_model qwen2.5-coder:7b
```

> Le nom doit être **exactement** celui affiché par `ollama list`
> (par exemple `qwen2.5-coder:7b`, avec les `:`).

Vérifie :

```bash
codewhale config list
```

Tu dois voir :

```text
default_text_model = qwen2.5-coder:7b
provider = ollama
```

---

## 8. Vérifier que tout fonctionne

La meilleure preuve : lancer une petite requête et regarder la réponse.

```bash
codewhale exec "Réponds exactement : OK"
```

- Si la réponse est `OK` → **tout marche**. 🎉
- Autre astuce : `codewhale doctor` affiche un diagnostic complet.

---

## 9. Basculer d'un modèle à l'autre

C'est **la seule commande à retenir**. Il suffit de changer le
`default_text_model` :

```bash
# Passer au modèle de raisonnement
codewhale config set default_text_model deepseek-r1:8b

# Revenir au modèle orienté code
codewhale config set default_text_model qwen2.5-coder:7b

# Utiliser le modèle alternatif DeepSeek-R1 (version HF)
codewhale config set default_text_model "hf.co/bartowski/DeepSeek-R1-Distill-Qwen-7B-GGUF:Q4_K_M"
```

Rappel : liste tes modèles disponibles avec `ollama list`, puis copie **le nom
exact** dans la commande.

> ⚠️ **Important** : le changement de modèle s'applique à la **prochaine
> session**. Ferme et relance Codewhale (ou démarre une nouvelle session) pour
> que le nouveau modèle soit actif.

### Comment choisir entre les modèles ?

- **`qwen2.5-coder:7b`** → spécialisé **code**. Le plus adapté pour un assistant
  qui écrit et modifie des programmes. ✅ Recommandé par défaut.
- **`deepseek-r1:8b`** → **raisonnement**. Il « réfléchit » avant de répondre ;
  utile pour des problèmes complexes, mais plus lent et plus verbeux.
- **`hf.co/bartowski/DeepSeek-R1-Distill-Qwen-7B-GGUF:Q4_K_M`** → variante de
  DeepSeek-R1, plus légère.

---

## 10. (Optionnel) Revenir aux modèles cloud

Si un jour tu veux réutiliser un modèle cloud, remets le provider d'origine :

```bash
codewhale config set provider deepseek
codewhale config set default_text_model deepseek-v4-pro
```

> 💡 Pour le cloud, il faudra aussi configurer une **clé API**
> (`codewhale login`). Avec Ollama local, **aucune clé n'est nécessaire**.

---

## 11. Dépannage

### « Je ne vois pas de réponse » ou « connexion refusée »

Le serveur Ollama n'est pas lancé. Dans un terminal :

```bash
ollama serve
```

puis reteste.

### « model not found »

Le modèle n'est pas téléchargé, ou le nom est mal écrit. Vérifie :

```bash
ollama list
```

et recopie **exactement** le nom (avec les `:` et majuscules éventuelles).

### « codewhale: command not found »

Codewhale n'est pas dans ton PATH. Réinstalle-le (étape 3) ou trouve-le avec :

```bash
which codewhale
```

### La config ne semble pas prise en compte

Relance Codewhale (nouvelle session). Le changement de provider/modèle ne
s'applique qu'aux **nouvelles** sessions.

### Voir où est le fichier de configuration

```bash
codewhale config path
```

Il affiche le chemin du fichier (généralement `~/.codewhale/config.toml`). Tu
peux aussi l'éditer à la main, mais les commandes `codewhale config set` sont
plus sûres.

---

## 12. Fiche mémo (anti-sèche)

```bash
# — Ollama —
ollama serve                                     # lancer le serveur
ollama pull qwen2.5-coder:7b                    # télécharger un modèle
ollama list                                      # lister les modèles
curl http://localhost:11434/api/tags             # vérifier que le serveur tourne

# — Codewhale : configurer Ollama —
codewhale config set provider ollama             # utiliser Ollama
codewhale config set providers.ollama.base_url http://localhost:11434

# — Codewhale : choisir / changer de modèle —
codewhale config set default_text_model qwen2.5-coder:7b   # modèle code
codewhale config set default_text_model deepseek-r1:8b     # modèle raisonnement

# — Vérifier —
codewhale config list                            # voir la config
codewhale doctor                                  # diagnostic complet
codewhale exec "Réponds exactement : OK"          # test réel
```

---

## Résumé de la configuration réalisée sur cette machine

État actuel (au moment de la rédaction) :

- **Serveur Ollama** : `http://localhost:11434`, version `0.23.1` (GPU CUDA).
- **Modèles téléchargés** : `qwen2.5-coder:7b`, `deepseek-r1:8b`,
  `hf.co/bartowski/DeepSeek-R1-Distill-Qwen-7B-GGUF:Q4_K_M`.
- **Codewhale** : `provider = ollama`,
  `default_text_model = qwen2.5-coder:7b`.
- **Vérifié** : `codewhale exec` répond correctement via Ollama. ✅
