# Organisation des fichiers d'un projet Godot 4

> Référence pour `dodge-the-creeps`, réutilisable pour les projets suivants.
> Écrit pour **Godot 4.4+** (fichiers `.uid`). Les numéros de phase et d'étape renvoient à [`prochaines-etapes.md`](../prochaines-etapes.md) ; le rangement lui-même y est l'étape 0.2.

## En bref

- **Ranger par feature** : la scène, son script et ses assets vivent dans le même dossier.
- **Le rôle d'un fichier se lit dans le type de sa classe** (`Resource`, `Node`, `RefCounted`, autoload), pas dans son dossier.
- **Tout en `snake_case`**, sauf les `class_name` en `PascalCase`.
- **Créer un dossier partagé (`common/`, `autoload/`…) au premier fichier qui en a besoin**, pas avant.
- **Déplacer et renommer depuis le dock FileSystem de Godot**, de préférence à l'explorateur ou à un `mv`.

---

## 1. Le principe : par feature, pas par couche

En Spring/Quarkus, on range **par couche** : `controller/`, `service/`, `dao/`, `dto/`. Godot fait l'inverse : on range **par feature** (*package-by-feature*). La scène, son script et ses assets vivent dans le même dossier.

Une scène (`.tscn`, fichier texte) embarque déjà la vue, la logique et ses ressources : c'est l'équivalent d'un composant Angular, dont le `.ts`, le `.html` et le `.scss` sont côte à côte. Séparer `player.tscn`, `player.gd` et `player.png` dans `scenes/`, `scripts/` et `sprites/` éclaterait un seul objet en trois endroits. La doc officielle recommande de grouper les assets au plus près des scènes, y compris les assets tiers d'un personnage, qui vont avec ses scènes et ses scripts.

| | Backend (Spring / Quarkus) | Godot |
|---|---|---|
| Axe de rangement | la couche technique | la feature (`player/`, `mob/`, `ui/`) |
| Unité de base | la classe | la scène : vue + logique + assets |
| Espaces de noms | packages | aucun : les `class_name` sont globaux |
| Séparation des responsabilités | par dossier | par **type de classe** (voir §2) |

---

## 2. Où passe la séparation des responsabilités

Le dossier regroupe par feature ; le rôle d'un fichier se lit dans ce que sa classe étend.

| Rôle (backend) | Équivalent Godot | Où le ranger |
|---|---|---|
| DTO / entité | `Resource` (sérialisable, champs `@export`). Pour de la donnée purement en mémoire : `RefCounted` (objet non-nœud, libéré automatiquement, l'équivalent d'un POJO ; c'est ce qu'est un script sans `extends`) | dans la feature (`mob/mob_type.gd`) |
| Ligne en base / config | un fichier `.tres` (instance d'une Resource, sauvegardée en texte) | à côté de son schéma (`mob/types/`) |
| Service sans état | `class_name` + `static func` : pas d'autoload | dans la feature qui l'utilise, sinon `common/` |
| Service singleton avec état | Autoload | `autoload/` |
| DAO / repository | l'autoload `SaveState`, qui lit et écrit le JSON avec `FileAccess` dans `user://` | `autoload/save_state.gd` ; les objets de données sérialisés (`SaveData`, étape 7.1) dans `save/` |
| Contrôleur | script racine de la scène, à garder fin | avec sa scène |
| Event bus / listener | signaux, plus un autoload `Events` s'il doit être global | `autoload/events.gd` |
| Composant réutilisable (Angular) | scène ou nœud composant | `common/components/` |
| Injection de dépendances | `@export` : la référence est branchée dans l'Inspecteur | avec la scène qui l'utilise |

**Pas de packages.** Les `class_name` sont globaux au projet et les dossiers ne créent aucun espace de noms. Deux `class_name Spawner` dans deux dossiers entrent en conflit : nomme précisément (`MobSpawner`, `PowerupSpawner`). Pour un pseudo-espace de noms, une classe interne s'utilise sous la forme `MobSpawner.Entry`.

---

## 3. Conventions de nommage

| Élément | Convention | Exemple |
|---|---|---|
| Dossiers et fichiers (`.gd`, `.tscn`, `.tres`, assets) | `snake_case` | `mob_spawner.gd`, `hud.tscn` |
| `class_name` | `PascalCase` | `MobSpawner` |
| Nœuds dans une scène | `PascalCase` | `MobTimer` |
| Variables, fonctions | `snake_case` | `spawn_mob()` |
| Signaux | `snake_case`, au passé | `score_changed`, `door_opened` |
| Constantes, valeurs d'enum | `CONSTANT_CASE` | `MAX_SPEED` |
| Membres privés (convention) | préfixe `_` | `_current_state` |
| Autoload (nom global) | `PascalCase`, sans `class_name` du même nom dans le script | `SaveState` |

- **Nom de fichier = nom de classe en `snake_case`** : `class_name MobSpawner` donne `mob_spawner.gd`. Une scène et son script portent le même nom, dans le même dossier : `player.tscn` + `player.gd`.
- **Casse.** L'éditeur sous Windows ne distingue pas `Player.gd` de `player.gd`, mais le jeu exporté si. Tout en `snake_case` évite le bug qui n'apparaît qu'après l'export.
- **Assets du tuto à renommer.** Ils ne respectent pas la convention : `playerGrey_up1.png` devient `player_grey_up1.png`, `House In a Forest Loop.ogg` devient `house_in_a_forest_loop.ogg`. Renomme-les depuis le dock FileSystem (`F2`) pour que les références suivent. Garde à côté de la police son fichier de licence.

---

## 4. Arborescence cible de `dodge-the-creeps`

Ce n'est pas un plan à créer d'un coup : chaque dossier apparaît à la phase qui en a besoin (voir §10).

- `[1.1]` : étape de la roadmap où le fichier apparaît ; `[Phase N]` quand il sert à toute la phase. `(ex.)` : nom indicatif.
- `res://` désigne la racine du projet (là où vit `project.godot`). `user://` désigne le dossier de données du joueur, hors projet.

```text
dodge-the-creeps/            # racine du dépôt = res://
│
├── project.godot            # config : scène principale, Input Map, autoloads, layers…
├── icon.svg                 # icône du projet
├── export_presets.cfg       # presets d'export (les secrets sont stockés ailleurs, cf. §6)
├── default_bus_layout.tres  # bus audio (Master / Music / SFX), créé quand tu modifies les bus
├── README.md
├── .gitignore               # au minimum : .godot/ (cache) et build/ (exports)
├── .gitattributes           # fins de ligne normalisées (LF)
├── .editorconfig            # (optionnel) impose les tabulations si tu édites les .gd hors de Godot
│
├── main/                    # point d'entrée : assemble les autres features
│   ├── main.tscn
│   ├── main.gd
│   └── score_keeper.gd      # [3.1] class_name ScoreKeeper : un nœud de Main, PAS un autoload
│
├── player/
│   ├── player.tscn
│   ├── player.gd
│   ├── sprites/             # player_grey_up1.png, _up2, _walk1, _walk2 (+ leurs .import)
│   └── states/              # [Phase 4] player_state.gd, normal_state.gd, shielded_state.gd… (ex.)
│
├── mob/
│   ├── mob.tscn
│   ├── mob.gd
│   ├── mob_type.gd          # [1.1] class_name MobType extends Resource : le schéma
│   ├── mob_spawner.gd       # [1.2] class_name MobSpawner : la Factory
│   ├── mob_pool.gd          # [6.1] après une mesure au profiler, remesurée ensuite
│   ├── types/               # [1.1] les .tres : données partagées (Flyweight)
│   │   ├── walk.tres        # nommés d'après l'animation qu'ils portent
│   │   ├── fly.tres
│   │   ├── swim.tres
│   │   └── fast.tres        # l'ancien fastmob.tscn, devenu une donnée
│   ├── behaviors/           # [Phase 2] mob_behavior.gd (base) + chase, zigzag…
│   └── sprites/             # enemy_flying_alt_1.png … (6 fichiers + leurs .import)
│
├── powerup/                 # [4.2] un bonus ramassable est une feature à part
│   ├── powerup.tscn
│   ├── powerup.gd
│   ├── powerup_type.gd      # Resource : schéma d'un bonus
│   └── types/               # shield.tres, slow_time.tres… (ex.)
│
├── wave/                    # [5.1] difficulté progressive (WaveData)
│   ├── wave_data.gd         # Resource : cadence, poids des mobs, durée…
│   ├── wave_director.gd     # lit les WaveData et pilote le MobSpawner (ex.)
│   └── waves/               # wave_01.tres, wave_02.tres…
│
├── save/                    # [7.1] les données sérialisées, sans état global
│   └── save_data.gd         # class_name SaveData extends RefCounted : le Memento
│
├── ui/
│   ├── hud.tscn
│   ├── hud.gd
│   └── theme/               # main_theme.tres : le Theme partagé par toute l'UI
│
├── autoload/                # globaux (Project Settings → Globals → Autoload), 2 à 4 au plus
│   ├── game_state.gd        # l'actuel GameState.gd, renommé en 0.2 ; machine à états en 4.1
│   ├── save_state.gd        # [3.1] highscore et sauvegarde, sortis de GameState
│   └── events.gd            # [Phase 3] bus de signaux global, seulement s'il se justifie
│
├── common/                  # créé au premier fichier réellement partagé
│   └── components/          # ex. state_machine.gd si Player ET Mob l'utilisent
│
├── audio/                   # médias partagés
│   ├── music/               # house_in_a_forest_loop.ogg
│   └── sfx/                 # gameover.wav
├── fonts/                   # xolonium_regular.ttf + son fichier de licence
├── shaders/                 # (si besoin) .gdshader partagés
│
├── tests/                   # [7.2] miroir des features : tests/mob/test_mob_spawner.gd
├── docs/                    # notes de conception
│   └── .gdignore            # fichier vide : Godot n'importe rien de ce dossier
├── build/                   # sorties d'export (.exe, .pck…), ignoré par Git
│   └── .gdignore
└── addons/                  # [7.2] gdUnit4, licence incluse ; l'outillage d'éditeur reste ignoré (cf. §6)
```

Générés ou hors dépôt :

```text
.godot/  # cache d'import et état de l'éditeur : régénéré, jamais versionné
user://  # sauvegardes et réglages du joueur : hors du projet (cf. §6)
```

**Lecture de l'arbre**

- **Features** (`main/`, `player/`, `mob/`, `powerup/`, `wave/`, `save/`, `ui/`) : chacune est autonome, avec ses scènes, scripts, sprites et données. Les sous-dossiers par rôle (`sprites/`, `states/`, `types/`) restent à l'intérieur de la feature : ce n'est pas un rangement par couche.
- **Partagé** (`common/`, `audio/`, `fonts/`, `shaders/`) : uniquement ce qui sert à deux features ou plus.
- **Globaux** (`autoload/`) : tous les singletons tiennent dans un seul dossier, ce qui les garde visibles, donc rares.
- **Données** : le schéma (`.gd`) et ses instances (`.tres`) restent ensemble. `types/` et `waves/` ne contiennent que des `.tres`, pas de code.
- **Hors jeu** (`docs/`, `tests/`, `build/`) : `docs/` et `build/` portent un `.gdignore`. Pour garder `tests/` et `addons/` hors de l'export, ajoute-les dans les filtres d'exclusion du preset (Export → onglet Resources → « Filters to exclude files/folders from project »).
- **Strategy** : la classe de base d'un comportement peut être marquée `@abstract` (Godot 4.5+).

---

## 5. Ce qu'il y a vraiment sur le disque

Un dossier de feature contient plus de fichiers que ce que montre le dock FileSystem. Zoom sur `mob/`, tel que `git status` le voit :

```text
mob/
├── mob.tscn
├── mob.gd
├── mob.gd.uid                         # UID du script (Godot 4.4+) : à commiter avec lui
├── mob_type.gd
├── mob_type.gd.uid
├── types/
│   ├── fly.tres                       # pas de fichier annexe : l'UID est dans l'en-tête du .tres
│   └── …
└── sprites/
    ├── enemy_flying_alt_1.png
    ├── enemy_flying_alt_1.png.import  # réglages d'import de l'asset : à commiter
    └── …
```

- **`.gd.uid`** : identifiant unique (UID) et stable du script, généré automatiquement depuis Godot 4.4. À commiter avec le script, y compris dans un commit de rangement.
- **`.import`** : réglages d'import d'un asset (compression, filtrage…). À commiter. Le résultat converti, lui, vit dans `.godot/imported/` et n'est jamais versionné.
- **`.tscn` et `.tres`** : pas de fichier annexe, leur UID est écrit dans leur en-tête.
- Tous ces fichiers suivent leur propriétaire quand tu déplaces celui-ci depuis le dock.

---

## 6. La racine du projet : que versionner ?

| Élément | Rôle | Versionné |
|---|---|---|
| `project.godot` | configuration du projet (fichier texte) | oui |
| `*.import` | réglages d'import d'un asset, posé à côté de lui | oui |
| `*.gd.uid` | UID stable d'un script (Godot 4.4+) | oui |
| `export_presets.cfg` | presets d'export | oui, en général : les secrets (mots de passe, clés) sont stockés à part, dans `.godot/export_credentials.cfg` (certains modèles de `.gitignore` l'excluent quand même) |
| `default_bus_layout.tres` | bus audio | oui |
| `.gitattributes` | fins de ligne (LF), utile sous Windows | oui |
| `.gdignore` | fichier vide qui écarte un dossier de l'import | oui |
| `addons/` | plugins tiers | oui pour ceux dont le jeu ou les tests ont besoin (gdUnit4), avec leurs licences ; **non** pour l'outillage d'éditeur à binaires natifs, réinstallable depuis l'AssetLib (`godot-git-plugin` : 25 Mo pour trois OS) |
| `.godot/` | cache d'import et état de l'éditeur, régénérés | non |
| `build/` | sorties d'export | non |
| `user://` | données du joueur à l'exécution | hors projet, hors dépôt |

- **Créer un `.gdignore` sous Windows.** L'Explorateur refuse un nom qui commence par un point : nomme le fichier `.gdignore.` (le point final disparaît à la validation), ou tape `touch docs/.gdignore` dans Git Bash.
- **Retrouver `user://`.** Sous Windows : `%APPDATA%\Godot\app_userdata\<nom-du-projet>\`. Depuis l'éditeur : Project → Open User Data Folder. C'est là que vit ton meilleur score.
- **Exporter.** Exporte dans `build/` (ignoré par Git, avec un `.gdignore` pour que Godot ne le scanne pas) ou hors du projet.

---

## 7. Grossir sans se noyer

Repères, pas des lois :

- Reste à plat tant que la racine tient sur un écran (une dizaine de dossiers).
- Au-delà, ajoute **un seul** niveau de regroupement thématique (`characters/`, `combat/`, `world/`), jamais un regroupement par type de fichier.
- Un dossier de feature qui dépasse une quinzaine de fichiers se découpe d'abord en sous-dossiers par rôle (`sprites/`, `states/`, `types/`), avant de se scinder en deux features.
- Trois niveaux maximum sous la racine : `mob/types/fly.tres` est la limite.

Exemple indicatif pour le futur RPG au tour par tour, à redessiner quand le projet démarrera :

```text
pen-and-paper-rpg/                # nom fictif
├── project.godot, .gitignore, …  # même racine que ci-dessus
│
├── main/                         # boot, changement de scène
├── characters/                   # regroupement thématique (un seul niveau)
│   ├── party/                    # party_member.tscn, character_class.gd, classes/*.tres
│   └── enemies/                  # enemy.tscn, enemy_data.gd, data/*.tres
├── combat/                       # combat au tour par tour
│   ├── combat.tscn
│   ├── combat.gd
│   ├── turn_manager.gd
│   ├── actions/                  # attack, spell, item, defend, flee
│   └── ui/                       # menu d'actions, barres de vie : propres au combat, donc ici
├── world/                        # exploration, cartes, dialogues
├── items/                        # item_data.gd + data/*.tres
├── spells/                       # spell_data.gd + data/*.tres
├── ui/                           # écrans génériques : menu principal, options, thème
├── save/
├── autoload/
├── common/
├── audio/   fonts/   shaders/
├── art_source/                   # .aseprite, .psd… avec un .gdignore : Godot ne les scanne pas
├── translations/                 # CSV de traduction, si tu localises
├── tests/
└── addons/
```

---

## 8. « Où je mets ce fichier ? »

1. **Propre à une seule feature ?** Dans son dossier, avec un sous-dossier par rôle si ça déborde. Vaut aussi pour son UI spécifique et pour les assets tiers qui lui sont liés.
2. **Partagé par deux features ou plus ?** Code ou composant : `common/`. Média : `audio/`, `fonts/`, `shaders/`.
3. **Global à toute la partie, avec un état ?** Autoload, dans `autoload/`. Sans état : `static func`, pas d'autoload.
4. **Du contenu (un mob, un bonus, une vague) ?** Un `.tres` à côté de son schéma (`types/`, `waves/`), jamais dans un `resources/` global.
5. **Un écran générique (menu, options, thème) ?** `ui/`.
6. **Un plugin tiers ?** `addons/`, avec sa licence.
7. **Pas dans le jeu (notes, sources d'art, exports) ?** Un dossier avec `.gdignore`, ou hors du projet.

---

## 9. Anti-patterns

| Piège | Pourquoi c'est un problème | À la place |
|---|---|---|
| `scripts/`, `scenes/`, `sprites/` à la racine | éclate chaque entité en trois dossiers ; Godot l'autorise, mais la doc recommande l'inverse | un dossier par feature |
| `common/` ou `utils/` créé d'avance | devient le fourre-tout, comme un `utils/` Java | le créer au premier fichier réellement partagé |
| `resources/` global pour tous les `.tres` | package-by-type déguisé : les données s'éloignent de leur schéma | `types/` à côté du schéma |
| Beaucoup d'autoloads | état global mutable, donc couplage | 2 à 4, et des signaux ou `@export` pour le reste |
| `class_name` identique au nom d'un autoload | Godot refuse : la classe masque le singleton (ex. `class_name ScoreKeeper` enregistré en autoload `ScoreKeeper`) | des noms distincts, ou pas de `class_name` dans un script d'autoload |
| Chemins `res://…` écrits en dur | à vérifier à la main après un déplacement | `@export var x: PackedScene` |
| Casse mélangée (`Player.gd`) | passe dans l'éditeur, casse à l'export | tout en `snake_case` |
| Plus de trois niveaux de dossiers | navigation pénible, chemins interminables | aplatir |
| Committer `.godot/` | cache lourd et régénéré | l'ignorer |
| Déplacer via l'explorateur ou `mv` | Godot retrouve souvent le fichier par son UID, sans garantie (chemins en dur, `.import` ou `.uid` oubliés) | le dock FileSystem |

```gdscript
# Fragile : chemin en dur, à vérifier à la main après un déplacement
var hardcoded_scene: PackedScene = preload("res://mob/mob.tscn")

# Robuste : la référence est portée par la scène (Inspecteur), l'éditeur la met à jour
@export var mob_scene: PackedScene
```

---

## 10. Quand chaque dossier apparaît, et plan de rangement

Un commit par étape, nommé d'après le pattern : le dossier créé à cette étape arrive dans le même commit.

| Moment | Ce qui apparaît |
|---|---|
| 0.2 — rangement (commit dédié, sans code) | `main/`, `player/`, `mob/`, `ui/`, `autoload/` (avec `game_state.gd`), `audio/`, `fonts/` ; assets renommés en `snake_case` |
| 1.1 — `MobType` en Resource | `mob/mob_type.gd`, `mob/types/*.tres` |
| 1.2 — `MobSpawner` (Factory) | `mob/mob_spawner.gd` |
| Phase 2 — Strategy | `mob/behaviors/` |
| 3.1 — Observer sur le score | `main/score_keeper.gd`, `autoload/save_state.gd` ; `autoload/events.gd` seulement si un bus global se justifie, sinon des signaux directs |
| 4.1 — FSM globale | rien de nouveau : `autoload/game_state.gd` change de rôle |
| 4.2 — power-up en State | `player/states/`, `powerup/` ; `common/components/state_machine.gd` si la machine est partagée |
| 5.1 — `WaveData` | `wave/` |
| 6.1 — Object Pool, après mesure | `mob/mob_pool.gd` (à déplacer dans `common/` dès qu'un second fichier l'utilise) |
| 6.2 — fond défilant | `background/` (ex.), une feature à part |
| 7.1 — Memento | `save/save_data.gd` |
| 7.2 — tests | `tests/`, `addons/gdUnit4/` |

**Le commit de rangement, pas à pas**

Correspondance avec le dépôt tel qu'il est (vérifiée sur `git ls-files` le 24 septembre 2026) :

| Aujourd'hui | Après rangement |
|---|---|
| `main.tscn`, `main.gd` | `main/` |
| `player.tscn`, `player.gd` | `player/` |
| `mob.tscn`, `mob.gd` | `mob/` |
| `hud.tscn`, `hud.gd` | `ui/` |
| `GameState.gd` (+ `.uid`) | `autoload/game_state.gd` ; l'autoload est déclaré par UID, le nom global `GameState` ne change pas |
| `art/playerGrey_*.png` | `player/sprites/player_grey_*.png` |
| `art/enemy*.png` | `mob/sprites/enemy_*.png` (ex. `enemyFlyingAlt_1.png` devient `enemy_flying_alt_1.png`) |
| `art/House In a Forest Loop.ogg` | `audio/music/house_in_a_forest_loop.ogg` |
| `art/gameover.wav` | `audio/sfx/gameover.wav` |
| `fonts/Xolonium-Regular.ttf` | `fonts/xolonium_regular.ttf` (`LICENSE.txt` et `FONTLOG.txt` restent à côté) |
| tout autre script | selon son rôle (voir §8) |

1. Crée une branche : `git switch -c refactor/structure-par-feature`.
2. Dans le dock FileSystem : crée les dossiers, puis glisse chaque `.tscn` avec son `.gd` (les `.gd.uid` et les `.import` suivent).
3. Renomme les assets en `snake_case` depuis le dock (`F2`).
4. Lance le jeu (`F5`), puis cherche `res://` dans tout le projet (`Ctrl+Shift+F`) pour repérer les chemins écrits en dur.
5. Stage et contrôle :

```bash
git add -A
git status   # attendu : des « renamed: », pas des paires « deleted » / « new file »
```

6. Un seul commit, sans aucune modification de code, pour que le diff reste lisible, et qui ferme l'issue de l'étape : `git commit -m "refactor: structure par feature (etape 0.2)" -m "Closes #N"`, avec le numéro que te donne `prochaine-session.sh`. Puis `git push` (et la fusion de la branche si tu en as créé une).

---

## Sources

- [Project organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html) (doc officielle)
- [GDScript style guide](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html)
- [Singletons (Autoload)](https://docs.godotengine.org/en/stable/tutorials/scripting/singletons_autoload.html)
- [Version control systems](https://docs.godotengine.org/en/stable/tutorials/best_practices/version_control_systems.html)
