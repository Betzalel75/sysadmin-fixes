# Rapport de correctif — Skills `pptx` et `ui-ux-pro-max` invisibles dans DeepSeek Harness

**Date :** 20 octobre 2026
**Environnement :** DeepSeek Harness `0.2.1-rc.2` (Node.js v24.16.0), interface Web `http://127.0.0.1:3080`
**Périmètre :** `/home/capybara/.agents/skills/` — 20 entrées, dont 19 `SKILL.md`
**Statut :** corrigé et vérifié

---

## 1. Problème

L'utilisateur dispose de 20 skills dans `~/.agents/skills/`. Deux d'entre eux n'apparaissaient pas dans l'interface du Harness :

| Skill (dossier) | Attendu | Constaté |
|---|---|---|
| `pptx` | apparaît sous le nom `pptx` | invisible sous ce nom ; apparaissait sous `custom-pptx` |
| `ui-ux-pro-max` | apparaît dans le menu `/` | totalement absent, aucune trace |

Les 18 autres skills étaient correctement listés, y compris ceux marqués *user-only* (`commit-rules`, `developer-rules`, `hexsecgpt`, `post-linkedin-rules`, `rapport-dactivite`).

Le symptôme était donc **double** et non homogène : un problème de nommage pour l'un, un problème de parsing pour l'autre.

---

## 2. Causes

### 2.1. Cause n°1 — `pptx` : nom du frontmatter ≠ nom attendu

Le fichier `~/.agents/skills/pptx/SKILL.md` déclarait :

```yaml
name: custom-pptx
```

Or le chargeur de skills ignore le nom du dossier et fait foi sur le champ `name` du frontmatter :

```js
// @deepseek-ai/dsh-skill-filesystem, parseSkillFile()
const name = stringField(parsed.data, "name");
if (name === void 0 || description === void 0) return; // skill ignoré
if (!isSkillName(name)) return;                          // /^[a-z0-9]+(?:-[a-z0-9]+)*$/
```

`custom-pptx` est un nom valide : le skill était donc bien chargé, mais **enregistré et affiché sous `custom-pptx`**. L'utilisateur cherchant `pptx` ne le trouvait pas. Le nom du dossier ne servait qu'à localiser le fichier, pas à nommer le skill.

### 2.2. Cause n°2 — `ui-ux-pro-max` : frontmatter non détecté (ligne vide initiale)

Le fichier `~/.agents/skills/ui-ux-pro-max/SKILL.md` commençait par un retour à la ligne, puis `---` :

```
\n---\nname: ui-ux-pro-max\n...
```

Le détecteur de frontmatter exige que **la toute première ligne** du fichier soit exactement `---` :

```js
// @deepseek-ai/dsh-skill-filesystem, parseFrontmatter()
if (raw.slice(0, firstLineEnd).replace(/\r$/, "") !== "---") return void 0;
```

Un seul `\n` en tête suffit à invalider la détection. Conséquence en cascade :

1. `parseFrontmatter()` retourne `undefined` ;
2. `parseSkillFile()` journalise `missing YAML frontmatter` et retourne `undefined` ;
3. `discoverRoot()` n'ajoute aucun candidat ;
4. le skill n'existe ni dans le catalogue modèle, ni dans la liste `skills/list` de l'interface Web.

C'était le **seul** skill du répertoire dans cette situation — ce qui explique qu'il était le seul totalement absent, alors que les autres skills *user-only* restaient visibles.

### 2.3. Précision sur l'affichage « user-only »

Le menu `/` de l'interface liste les skills filtrés par `isUserInvocable` (`dsh-api-session-controller/lib/types/skill-catalog.js`), puis préfixe la description par `user-only` lorsque `modelInvocable` est faux (`dsh-client-ui-skill/lib/client.js`). Un skill marqué `disable-model-invocation: true` est donc **visible**, mais non auto-chargeable par le modèle. Ce mécanisme n'était pas en cause.

---

## 3. Solution appliquée

Deux corrections chirurgicales, sans modification du Harness ni de la configuration du profil.

### 3.1. `pptx` — alignement du nom

`~/.agents/skills/pptx/SKILL.md` :

```diff
  ---
- name: custom-pptx
+ name: pptx
  description: Create professional, editable PowerPoint presentations (.pptx) ...
  ---
```

Aucune autre occurrence du nom n'a été touchée : les références résiduelles dans le corps du fichier (`# custom-pptx — Professional Slide Deck Generator`, arborescence `custom-pptx/`, chemins `/home/z/my-project/...`) sont documentaires et sans effet sur le chargement.

### 3.2. `ui-ux-pro-max` — restauration du frontmatter

`~/.agents/skills/ui-ux-pro-max/SKILL.md` :

```diff
-
  ---
  name: ui-ux-pro-max
  description: A skill/workflow for AI coding assistants ...
  disable-model-invocation: true
  ---
```

Suppression du seul retour à la ligne initial. Le contenu, le nom et la politique d'invocation sont inchangés.

### 3.3. Vérification

1. **Rejeu des règles du chargeur** sur les 19 `SKILL.md` (détection frontmatter en première ligne, présence de `name` et `description`, conformité au motif kebab-case, cohérence nom/dossier) : **19/19 valides**.
2. **Rechargement à chaud** : le catalogue de la session a été réémis automatiquement, `custom-pptx` a disparu et `pptx` est apparu — preuve que le watcher `chokidar` du provider a bien repris le changement.
3. **`ui-ux-pro-max`** est désormais parsé et donc éligible à `skills/list` (invocation utilisateur par défaut : `true`). Un rafraîchissement de la page (F5) reste nécessaire, le menu mettant son catalogue en cache par session.

---

## 4. Points restants (non traités, hors demande)

| Point | Détail | Impact |
|---|---|---|
| `ui-ux-pro-max` en *user-only* | `disable-model-invocation: true` conservé (intention d'origine du skill) | visible dans le menu `/`, mais jamais auto-chargé par le modèle |
| `scripts/` sans `SKILL.md` | contient `generate_catalog.py`, `install_ctf_tools.sh`, `skill_security_auditor.py` | ne sera jamais listé comme skill |
| `pptx` porté d'un autre environnement | corps référençant `AskUserQuestion`, `z-ai`, `/home/z/my-project/...`, `batch_html2pptx.js` absent de `scripts/` | s'affiche, mais l'exécution échouera tant qu'il n'est pas adapté |

---

## 5. Recommandations de prévention

1. **Faire correspondre `name` et nom de dossier** pour tout skill : c'est la source de confusion la plus fréquente lors d'un import depuis un autre écosystème.
2. **Ne jamais laisser de ligne vide ou de BOM** avant le `---` d'ouverture ; c'est un échec silencieux (simple `warn` dans les logs), sans erreur visible dans l'interface.
3. **Ajouter un contrôle d'intégrité** au dépôt de skills (script de validation reprenant les règles du chargeur) pour détecter ces deux cas avant de lancer le Harness.
4. Pour diagnostiquer un skill manquant : vérifier l'ordre des causes — frontmatter détecté → `name`/`description` présents → nom kebab-case valide → politique d'invocation. Un skill peut être valide et simplement masqué (`user-only`) ou renommé.
