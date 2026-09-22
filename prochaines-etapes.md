# Roadmap — de l'analyse à la pratique

Huit jeux décortiqués, un référentiel de patterns, une référence de bugs, un glossaire, et une passe de vérification contre les désassemblages. La phase d'analyse est refermée : la suite se joue dans Godot.

## Le parti pris de cette roadmap

Dodge the Creeps est un **POC d'apprentissage**, pas un jeu à livrer. L'objectif n'est donc pas le code minimal qui fonctionne, mais la **surface de concepts touchée** : on y fait entrer tout ce qui peut y entrer honnêtement, y compris là où un vrai projet s'en passerait.

C'est un choix assumé, et il a une contrepartie qui fait partie de l'apprentissage. Chaque étape porte une ligne **« en vrai projet »** qui dit ce que le pattern coûterait si l'objectif était de livrer. Appliquer Strategy partout sans le savoir, c'est acquérir un réflexe ; l'appliquer partout en sachant que deux cas stables n'en avaient pas besoin, c'est acquérir un jugement. La deuxième compétence est celle qui sert.

Rythme visé : une étape par session, deux sessions par semaine. Soit environ sept semaines pour les phases 0 à 7, les phases 8 et 9 étant du bonus.

**Une règle de méthode pour toute la roadmap** : un commit par étape, avec un message qui nomme le pattern (`feat: comportements de mob interchangeables (Strategy)`). L'historique devient le journal de la montée en compétence — et c'est ce qui manquait aux trois premiers commits du dépôt.

---

## Phase 0 — assainir le terrain

### Étape 0.1 — typage statique partout

Le code actuel est non typé (`func update_high_score(score):`, `var file = ...`). Godot accepte, mais le typage statique attrape à l'analyse ce qui sinon plante à l'exécution, et il débloque l'autocomplétion.

```gdscript
# Avant
func update_high_score(score):
	if score > high_score:

# Après
func update_high_score(score: int) -> void:
	if score > high_score:
```

À faire sur les cinq scripts. Rien ne change à l'écran — et c'est justement l'exercice : un refactor à comportement identique, vérifiable par le simple fait que le jeu tourne pareil.

**En vrai projet** : non négociable, on commence comme ça. C'est la seule étape de cette roadmap qui n'est pas un surdimensionnement.

**Écho au corpus** : les trois bugs de [Final Fantasy](./final-fantasy-1) sont des confusions de champ entre deux valeurs du même type. Le typage seul ne les aurait pas attrapés (un index et un taux sont tous deux des `int`), mais c'est la première ligne de défense de la famille.

---

## Phase 1 — les données avant le comportement

L'ordre compte : on modélise avant de faire varier. C'est le niveau 3 de la grille avant le niveau 4.

### Étape 1.1 — `MobType` en Resource

Aujourd'hui `mob.gd` tire une animation au hasard parmi trois, et `fastmob.tscn` est une copie de scène. Les deux sont à remplacer par **une** scène paramétrée par une donnée.

```gdscript
class_name MobType
extends Resource

@export var display_name: String = ""
@export var sprite_frames: SpriteFrames
@export var animation: StringName = &"walk"
@export_range(50.0, 600.0) var speed_min: float = 150.0
@export_range(50.0, 600.0) var speed_max: float = 250.0
@export var score_value: int = 1
```

Volontairement **pas** de champ `behavior` ici : il arrive à l'étape 2.1. Un `@export` typé sur une classe qui n'existe pas encore empêche le script de parser — Godot ne compile pas une déclaration dont il ne connaît pas le type. C'est le premier endroit où l'ordre des étapes n'est pas cosmétique.

Puis `mob.gd` référence son type au lieu de porter ses valeurs :

```gdscript
extends RigidBody2D

var type: MobType

func setup(p_type: MobType) -> void:
	type = p_type
	$AnimatedSprite2D.sprite_frames = type.sprite_frames
	$AnimatedSprite2D.animation = type.animation
	$AnimatedSprite2D.play()
```

Créer trois `.tres` : `mob_walk.tres`, `mob_fly.tres`, `mob_swim.tres`. **Supprimer `fastmob.tscn`** — il devient un quatrième `.tres` avec une vitesse plus haute, pas une scène.

**En vrai projet** : justifié dès trois variantes. C'est le pattern le plus rentable du corpus.

**Écho au corpus** : `PokemonSpecies` / `Pokemon` ([Pokémon](./pokemon-rouge-bleu), niveau 3) et `MonsterType` ([Zelda 1](./zelda-1), niveau 3). Composition over inheritance et Flyweight en même temps — la Resource chargée depuis un chemin est partagée par toutes les instances qui la citent.

**Le piège à voir en vrai** : écrire `type.speed_max = 400` depuis `mob.gd` corrompt la donnée de référence pour **tous** les mobs de ce type, tant que le jeu tourne. C'est exactement `target.species.base_defense -= 10` de l'analyse Pokémon. À tester une fois volontairement, pour le voir.

### Étape 1.2 — `MobSpawner` et la Factory

Sortir la construction du mob de `main.gd`, qui en fait trop.

```gdscript
class_name MobSpawner
extends Node

@export var mob_scene: PackedScene
@export var available_types: Array[MobType] = []

func spawn_at(spawn_point: PathFollow2D) -> RigidBody2D:
	var type: MobType = available_types.pick_random()
	var mob := mob_scene.instantiate() as RigidBody2D
	mob.setup(type)
	spawn_point.progress_ratio = randf()
	mob.position = spawn_point.position
	var direction := spawn_point.rotation + PI / 2 + randf_range(-PI / 4, PI / 4)
	mob.rotation = direction
	var speed := randf_range(type.speed_min, type.speed_max)
	mob.linear_velocity = Vector2(speed, 0.0).rotated(direction)
	return mob
```

`main.gd` se réduit à `add_child(spawner.spawn_at($MobPath/MobSpawnLocation))`.

**En vrai projet** : justifié. Une fabrique isole la construction, et c'est elle qu'on remplace par un pool à la phase 6 sans toucher aux appelants.

**Écho au corpus** : `Pokemon.from_species()`, et la variante un-vers-plusieurs de [Mario](./super-mario-bros) où un `LevelObject` produit N cellules.

---

## Phase 2 — le comportement interchangeable

### Étape 2.1 — Strategy à comportement identique

L'étape la plus formatrice de la roadmap, parce que **rien ne doit changer à l'écran**. On extrait le comportement actuel dans une classe, on le branche par la donnée, et le jeu se joue exactement pareil.

```gdscript
class_name MobBehavior
extends Resource

## Appelé une fois au spawn, après setup(). Le comportement par défaut
## est celui du jeu actuel : partir tout droit et ne plus rien décider.
##
## La vitesse est rangée sur le MOB et non sur ce Resource, parce que
## le Resource est PARTAGÉ entre tous les mobs de ce type : y stocker
## une valeur par instance les ferait tous bouger ensemble.
func on_spawn(mob: RigidBody2D, direction: float, speed: float) -> void:
	mob.set_meta(&"speed", speed)
	mob.linear_velocity = Vector2(speed, 0.0).rotated(direction)

## Appelé à chaque frame physique. Vide par défaut : un mob qui va
## tout droit n'a rien à faire, la vélocité initiale suffit.
func update(mob: RigidBody2D, delta: float) -> void:
	pass
```

`StraightBehavior` n'a donc **rien à écrire** — c'est la classe de base telle quelle. Créer une sous-classe vide juste pour lui donner un nom est une option ; s'en passer en est une autre, et c'est celle que je retiens.

Le `MobSpawner` délègue entièrement : **retire de `spawn_at()` la ligne qui pose `linear_velocity`** et remplace-la par `type.behavior.on_spawn(mob, direction, speed)`. Sans ça, deux endroits décident de la vélocité et le comportement n'a jamais la main. Et `mob.gd` appelle `type.behavior.update(self, delta)` dans `_physics_process`.

Test d'acceptation : le jeu est indiscernable de la version précédente.

### Étape 2.2 — les variantes qui prouvent que le pattern paie

```gdscript
class_name ChaseBehavior
extends MobBehavior

@export var turn_rate: float = 2.0

## Pas de on_spawn : celui de la classe de base pose déjà la vitesse
## et la vélocité initiale. Le redéfinir sans appeler super() est
## précisément l'erreur qui laisserait get_meta("speed") sans valeur.

func update(mob: RigidBody2D, delta: float) -> void:
	var player := mob.get_tree().get_first_node_in_group(&"player")
	if player == null:
		return
	var wanted := mob.global_position.direction_to(player.global_position)
	var current := mob.linear_velocity.normalized()
	var steered := current.lerp(wanted, turn_rate * delta).normalized()
	mob.linear_velocity = steered * mob.get_meta(&"speed")

class_name ZigzagBehavior
extends MobBehavior

@export var amplitude: float = 120.0
@export var frequency: float = 3.0

func update(mob: RigidBody2D, delta: float) -> void:
	var t: float = mob.get_meta(&"age", 0.0) + delta
	mob.set_meta(&"age", t)
	var forward := mob.linear_velocity.normalized()
	var lateral := forward.orthogonal() * sin(t * frequency) * amplitude
	mob.linear_velocity = forward * mob.get_meta(&"speed") + lateral
```

Ajouter un comportement = un fichier, zéro ligne touchée ailleurs. C'est la démonstration.

**En vrai projet** : justifié dès que le comportement varie, ce qui est le cas ici. La phase 2 est celle où le surdimensionnement s'arrête et où le pattern est simplement correct.

**Écho au corpus** : `MoveEffect` ([Pokémon](./pokemon-rouge-bleu)), `BeamEffect` ([Metroid](./metroid)), les profils de pilotage de [F-Zero](./fzero). Et le point commun des huit jeux : **le comportement est un champ de la donnée de référence**, éditable dans l'inspecteur, pas un branchement dans le moteur.

**Note de conception** : `set_meta` ci-dessus est un raccourci discutable — l'état du comportement vit sur le mob plutôt que dans le comportement. C'est volontaire, parce que la Resource est *partagée* entre tous les mobs de ce type : y stocker un `age` les ferait tous zigzaguer en phase. C'est le piège du Flyweight de l'étape 1.1, rencontré dans l'autre sens. La version propre serait un objet d'état par instance ; à discuter quand tu y arriveras.

---

## Phase 3 — découpler l'affichage

### Étape 3.1 — Observer sur le score

Aujourd'hui `main.gd` fait `$HUD.update_score(score)` : la logique de jeu connaît l'existence de l'interface, et son chemin dans l'arbre.

L'autoload actuel mélange deux responsabilités : l'état de jeu et la persistance. On les sépare ici — `SaveState` garde le highscore et la sauvegarde, `GameState` deviendra la machine à états à l'étape 4.1. Un autoload par responsabilité.

```gdscript
class_name ScoreKeeper
extends Node

signal score_changed(value: int)
signal high_score_beaten(value: int)

var _score: int = 0

var score: int:
	get: return _score

func add(points: int) -> void:
	_score += points
	score_changed.emit(_score)
	if _score > SaveState.high_score:
		high_score_beaten.emit(_score)

func reset() -> void:
	_score = 0
	score_changed.emit(_score)
```

Le HUD s'abonne, et `main.gd` ne le connaît plus.

**Deux pièges à rencontrer volontairement.** Brancher `high_score_beaten` sur une animation sans `CONNECT_ONE_SHOT` : elle se rejouera à chaque point au-delà du record. Et ne pas `disconnect()` en changeant de partie : les connexions s'accumulent, l'effet se déclenche N fois.

**En vrai projet** : justifié, et c'est même le minimum. Un objet de données qui connaît l'UI est une inversion de couche.

**Écho au corpus** : [Pokémon](./pokemon-rouge-bleu) niveau 4, avec le cas qui monte en difficulté chez [Final Fantasy](./final-fantasy-1) — quatre barres de PV et huit compteurs à suivre rendent l'appel direct intenable.

---

## Phase 4 — l'état, à deux échelles

### Étape 4.1 — la FSM globale que les originaux n'avaient pas

C'est le niveau 1 du corpus, et sa conclusion la plus contre-intuitive : **chez Pokémon, l'état global n'est pas une valeur unique** mais un ensemble de drapeaux posés par trois routines différentes, dont une seule les remet à zéro — d'où le glitch Trainer-Fly et son menu qui ne répond plus. C'est vérifié pour ce jeu ; pour les autres du corpus, c'est plausible mais non établi.

```gdscript
extends Node                          ## autoload GameState

enum State { TITLE, PLAYING, GAME_OVER }

signal state_changed(from: State, to: State)

var _state: State = State.TITLE

var state: State:
	get: return _state

## Un seul point d'écriture. Jamais de drapeau parallèle.
func transition_to(next: State) -> void:
	if next == _state:
		return
	var previous := _state
	_state = next
	state_changed.emit(previous, next)
```

Puis retirer tout ce qui dupliquait cette information ailleurs.

**En vrai projet** : justifié. C'est gratuit en Godot et ça élimine une famille entière de bugs.

**Écho au corpus** : le glitch **Trainer-Fly** de [Pokémon](./pokemon-rouge-bleu) — trois drapeaux indépendants, une seule routine pour les remettre à zéro, et un menu qui ne répond plus.

### Étape 4.2 — le power-up en State, avec la leçon de Sonic

Un power-up temporaire (bouclier, vitesse) est le cas où State devient **obligatoire** plutôt que décoratif, parce que l'état change une **dimension** du personnage.

```gdscript
class_name PlayerState
extends RefCounted

func speed() -> float:
	return 400.0

func is_invulnerable() -> bool:
	return false

func modulate_color() -> Color:
	return Color.WHITE

class_name ShieldedState
extends PlayerState

func is_invulnerable() -> bool:
	return true

func modulate_color() -> Color:
	return Color(0.6, 0.8, 1.0)

class_name HastedState
extends PlayerState

func speed() -> float:
	return 700.0

func modulate_color() -> Color:
	return Color(1.0, 0.9, 0.5)
```

Et le point qui fait tout, côté joueur :

```gdscript
## Fragile : chaque changement d'état doit penser à tout recopier.
func set_state(s: PlayerState) -> void:
	_state = s
	speed = s.speed()
	## ... et la couleur ? et l'invulnérabilité ?

## Robuste : rien à synchroniser, tout dérive de l'état courant.
func _process(delta: float) -> void:
	var velocity := _read_input() * _state.speed()
	$AnimatedSprite2D.modulate = _state.modulate_color()
```

**En vrai projet** : justifié dès que l'état change plus d'une valeur.

**Écho au corpus** : [Sonic](./sonic), le glitch du **capteur de poussée non repositionné en roulade**. Les rayons de collision changent entre debout et roulade, les capteurs de sol suivent, ceux de poussée non. C'est la seule famille de bug du corpus qui ne concerne aucune donnée invalide — juste un état partiellement appliqué. En dérivant les dimensions depuis l'état au lieu de les copier, il devient impossible à écrire.

---

## Phase 5 — la difficulté data-driven

### Étape 5.1 — `WaveData` et une vraie relation many-to-many

C'est l'étape où le niveau 3 du corpus se transpose vraiment : une relation avec payload, donc une **entité de jointure promue**.

```gdscript
class_name SpawnEntry
extends Resource

## L'entité de jointure : la relation WaveData ↔ MobType porte un poids,
## donc elle devient une entité à part entière.
@export var mob_type: MobType
@export_range(0, 100) var weight: int = 10

class_name WaveData
extends Resource

@export var label: String = ""
@export_range(0.0, 600.0) var starts_at_seconds: float = 0.0
@export_range(0.05, 3.0) var spawn_interval: float = 0.5
@export_range(0.5, 3.0) var speed_multiplier: float = 1.0
@export var entries: Array[SpawnEntry] = []

func pick_type(rng: RandomNumberGenerator) -> MobType:
	var total := 0
	for e in entries:
		total += e.weight
	var roll := rng.randi_range(1, maxi(1, total))
	for e in entries:
		roll -= e.weight
		if roll <= 0:
			return e.mob_type
	return entries[-1].mob_type
```

Quatre ou cinq `.tres` de paliers, et un `WaveController` qui sélectionne le palier courant selon le temps écoulé. **Dessine l'ERD** en Mermaid avant de coder — c'est l'exercice.

**En vrai projet** : justifié. Une courbe de difficulté en `.tres` se règle sans recompiler et sans toucher au code.

**Écho au corpus** : le movepool de [Pokémon](./pokemon-rouge-bleu) (jointure creuse → entité), la table de coûts de [Zelda II](./zelda-2) (jointure dense → matrice), les charges par niveau de [Final Fantasy](./final-fantasy-1) (choix du grain). Le critère est double : la relation porte-t-elle un attribut, et quel est son taux de remplissage ? Ici elle porte un poids et elle est creuse, donc entité de jointure.

---

## Phase 6 — mesurer avant d'optimiser

### Étape 6.1 — Object Pool, justifié par une mesure

L'étape où la méthode compte plus que le résultat. **Ordre imposé** :

1. Pousse le `spawn_interval` à `0.05` et laisse tourner jusqu'à 200 mobs à l'écran.
2. Ouvre le profiler de Godot (Debug → Profiler), regarde le temps passé dans l'instanciation et dans le GC.
3. **Note le chiffre.**
4. Implémente le pool.
5. Remesure, compare.

```gdscript
class_name MobPool
extends Node

@export var mob_scene: PackedScene
@export var size: int = 64

var _pool: Array[RigidBody2D] = []

func _ready() -> void:
	for i in size:
		var mob := mob_scene.instantiate() as RigidBody2D
		mob.hide()
		mob.set_physics_process(false)
		add_child(mob)
		_pool.append(mob)

func acquire() -> RigidBody2D:
	for mob in _pool:
		if not mob.visible:
			return mob
	return null                              ## pool épuisé : c'est une information

func release(mob: RigidBody2D) -> void:
	mob.hide()
	mob.set_physics_process(false)
	mob.linear_velocity = Vector2.ZERO
```

Il est possible que le gain soit **indétectable**. Ce serait un excellent résultat : tu auras appris à mesurer, et tu auras une anecdote chiffrée à opposer la prochaine fois qu'on te proposera d'optimiser à l'aveugle.

**En vrai projet** : injustifié tant que le profiler ne dit rien. Le critère n'est pas « beaucoup d'objets » mais « beaucoup d'objets **créés et détruits en rafale** ».

**Écho au corpus** : [Sonic](./sonic) et ses **32 anneaux dispersés dans une seule frame** — le seul cas légitime du corpus, et l'original les limite par le nombre de créneaux d'objets libres, pas par choix de design. Noter aussi que les anneaux ne testent le sol qu'**une frame sur quatre** : le pool est aussi le bon endroit pour distribuer la charge.

### Étape 6.2 — le fond défilant à bandes recyclées

Même principe que le pool, appliqué au décor plutôt qu'aux entités : **on ne crée rien, on recycle ce qui sort de l'écran.** C'est la seule façon honnête de toucher à la fenêtre glissante dans ce jeu, et ça remplace le `ColorRect` uni par quelque chose qui bouge.

Le fond est découpé en bandes horizontales qui descendent. Quand une bande sort par le bas, elle est repositionnée au-dessus de la plus haute, et **son contenu est redécidé à ce moment-là** — c'est exactement « décompresser juste en avance de ce qui est consommé ».

```gdscript
class_name ScrollingBackground
extends Node2D

## Bandes recyclees, comme Block_Buffer_1 et Block_Buffer_2 de
## Super Mario Bros. : celle qui sort de l'ecran est repositionnee
## devant l'autre, et son contenu est regenere a cet instant.

@export var band_textures: Array[Texture2D] = []
@export_range(10.0, 400.0) var scroll_speed: float = 60.0

var _bands: Array[Sprite2D] = []
var _band_height: float = 0.0

#region Mise en place
func _ready() -> void:
	assert(not band_textures.is_empty(), "il faut au moins une texture de bande")
	_band_height = float(band_textures[0].get_height())
	for t in band_textures:
		assert(float(t.get_height()) == _band_height,
			"toutes les bandes doivent avoir la meme hauteur")
	var count := _needed_band_count(get_viewport_rect().size.y)
	for i in count:
		var band := Sprite2D.new()
		band.centered = false
		band.texture = band_textures.pick_random()
		band.position = Vector2(0.0, -_band_height * i)
		add_child(band)
		_bands.append(band)

## Mario couvrait 32 colonnes de metatiles pour un ecran de 16 :
## une page visible, une page en avance. Meme invariant ici, sauf
## qu'on le calcule au lieu de le coder en dur.
func _needed_band_count(viewport_height: float) -> int:
	return ceili(viewport_height / _band_height) + 1
#endregion

func _process(delta: float) -> void:
	var viewport_height := get_viewport_rect().size.y
	for band in _bands:
		band.position.y += scroll_speed * delta
		if band.position.y >= viewport_height:
			_recycle(band)

func _recycle(band: Sprite2D) -> void:
	var highest := _bands[0]
	for b in _bands:
		if b.position.y < highest.position.y:
			highest = b
	band.position.y = highest.position.y - _band_height
	band.texture = band_textures.pick_random()
```

Trois choses à remarquer en le codant.

**Le nombre de bandes est un invariant, pas une constante.** `ceili(hauteur_écran / hauteur_bande) + 1` : assez pour couvrir l'écran, plus une en réserve. Coder `2` en dur marche tant que les bandes font la hauteur de l'écran, et casse silencieusement dès qu'on met une texture plus courte. C'est le genre d'hypothèse implicite qui produit les bugs du corpus.

**Le recyclage est le bon moment pour régénérer.** `band.texture = band_textures.pick_random()` est appelé exactement quand la bande devient invisible. Remplace `pick_random()` par une fonction de bruit indexée par la position et tu as une génération procédurale déterministe — la brique de la phase 9, en trois lignes.

**Le fond descend, donc le joueur monte.** Un fond qui défile change la lecture du jeu sans qu'une seule règle bouge : c'est de la donnée visuelle, pas un nouveau mode. La question du niveau 1, encore une fois.

**Bonus gratuit, la parallaxe** : deux instances de `ScrollingBackground` à des vitesses différentes, la plus lente derrière. Trois minutes de travail, et le premier effet de profondeur du projet.

**En vrai projet** : injustifié. Godot fournit `Parallax2D` (et l'ancien `ParallaxBackground`), qui fait le défilement et la répétition sans une ligne de code. Le coder à la main est **l'exercice**, pas la solution — et c'est précisément le genre de chose qu'il faut savoir avoir fait une fois pour comprendre ce que le nœud natif fait à ta place.

**Écho au corpus** : les deux `Block_Buffer` de [Mario](./super-mario-bros) (niveau 3), le chargement ligne par ligne de [Final Fantasy](./final-fantasy-1) (niveau 2), la fenêtre de 1024 × 1024 de [F-Zero](./fzero) (niveau 2). Trois axes différents — colonnes, lignes, les deux — et le même principe. Le tien glisse en lignes, comme Final Fantasy, parce que le jeu est en portrait et que le mouvement est vertical : **l'axe est toujours dicté par le mouvement, pas par le format.**

Et le piège de [F-Zero](./fzero) à connaître : sa carte **aliase tous les 1024 pixels**, parce qu'échantillonner au-delà de la fenêtre streamée renvoie ce qu'une autre portion a laissé dans la même cellule. Invisible en version commerciale, parce que la fenêtre est toujours plus large que le champ de vision. C'est l'argument du `+ 1` dans `_needed_band_count()` : la marge n'est pas du gaspillage, c'est ce qui rend l'erreur inatteignable.

---

## Phase 7 — persistance et tests

### Étape 7.1 — Memento, et le choix du grain

Étendre la sauvegarde au-delà du highscore : réglages de volume, meilleur score par type de mob, nombre de parties jouées.

```gdscript
class_name SaveData
extends RefCounted

const KEY_VERSION := "version"
const KEY_HIGH_SCORE := "high_score"
const KEY_GAMES_PLAYED := "games_played"
const KEY_VOLUME := "volume"

const CURRENT_VERSION := 1

var high_score: int = 0
var games_played: int = 0
var volume: float = 1.0

func to_dict() -> Dictionary:
	return {
		KEY_VERSION: CURRENT_VERSION,
		KEY_HIGH_SCORE: high_score,
		KEY_GAMES_PLAYED: games_played,
		KEY_VOLUME: volume,
	}

## Valider ce qui entre, champ par champ. Et couvrir EXACTEMENT
## le même ensemble de clés que to_dict().
func from_dict(data: Dictionary) -> void:
	high_score = int(data.get(KEY_HIGH_SCORE, 0))
	games_played = int(data.get(KEY_GAMES_PLAYED, 0))
	volume = clampf(float(data.get(KEY_VOLUME, 1.0)), 0.0, 1.0)
```

Le champ `version` est le détail qui vaut l'étape : il te permettra de migrer une sauvegarde existante quand tu ajouteras un champ, au lieu de la jeter.

**Question de grain à te poser explicitement** : faut-il stocker le score de **chaque** partie, ou seulement le meilleur et le compte ? Le second, et c'est la leçon de Zelda 1.

**En vrai projet** : justifié. Un champ de version dans une sauvegarde coûte deux lignes et évite de casser les parties des joueurs.

**Écho au corpus** : [Zelda 1](./zelda-1) — **un octet par salle**, avec un compteur de monstres **saturé** plutôt que la liste des morts. Le réflexe naturel serait de tout sérialiser ; c'est le piège. Et [Final Fantasy](./final-fantasy-1), bug **TMPR/SABR** : l'effet est calculé correctement puis jeté, parce que les deux routines de sérialisation ne couvrent pas le même ensemble de champs.

### Étape 7.2 — les tests unitaires

Installer [gdUnit4](https://github.com/MikeSchulze/gdUnit4) et écrire les quatre tests qui comptent :

```gdscript
extends GdUnitTestSuite

func test_aller_retour_de_sauvegarde() -> void:
	var original := SaveData.new()
	original.high_score = 4242
	original.games_played = 17
	original.volume = 0.5
	var relu := SaveData.new()
	relu.from_dict(original.to_dict())
	## Le test qui attrape le bug TMPR : la symétrie du couple.
	assert_int(relu.high_score).is_equal(4242)
	assert_int(relu.games_played).is_equal(17)
	assert_float(relu.volume).is_equal_approx(0.5, 0.001)

func test_sauvegarde_corrompue_ne_plante_pas() -> void:
	var s := SaveData.new()
	s.from_dict({})                          ## vide
	assert_int(s.high_score).is_equal(0)

func test_volume_est_borne() -> void:
	var s := SaveData.new()
	s.from_dict({SaveData.KEY_VOLUME: 99.0})
	assert_float(s.volume).is_equal_approx(1.0, 0.001)

func test_selection_de_type_respecte_les_poids() -> void:
	## 10 000 tirages, un type à poids 0 ne doit jamais sortir.
	var rng := RandomNumberGenerator.new()
	rng.seed = 12345
	var wave := load("res://data/waves/wave_1.tres") as WaveData
	## ... comptage et assertions
```

**En vrai projet** : justifié, et c'est l'étape que la plupart des projets hobby sautent.

**Écho au corpus** : c'est la conclusion opérationnelle de [`_framework/lecons-bugs.md`](./_framework/lecons-bugs.md). Les trois bugs de Final Fantasy sont les **seuls du corpus qu'un test unitaire aurait trouvés** — ils sont présents à chaque exécution, sur le chemin nominal. Les quatre autres familles exigent une manipulation ou une condition particulière. Savoir laquelle de tes fonctions tombe dans quelle catégorie, c'est savoir où les tests paient.

---

## Phase 8 — le bonus, si l'envie est là

### Étape 8.1 — Command et le remapping

```gdscript
class_name InputCommand
extends RefCounted

func execute(player: Node) -> void:
	pass

class_name MoveCommand
extends InputCommand

var direction: Vector2

func _init(dir: Vector2) -> void:
	direction = dir

func execute(player: Node) -> void:
	player.queue_move(direction)
```

Avec un `Dictionary` de liaisons, remapper une touche devient un échange d'entrée. Et l'historique des commandes est **gratuit** — ce qui donne un replay de la dernière partie, ou un mode fantôme.

**En vrai projet** : injustifié pour ce jeu. Justifié dès qu'il y a du remapping, de l'undo ou du replay.

**Écho au corpus** : [Zelda II](./zelda-2) le fait déjà sans le nommer, avec sa table de pointeurs de routines de sorts. Et c'est ce qui alimentera le journal de combat du RPG sans code dédié.

### Étape 8.2 — les déblocages en bitmask

Des succès ou des types de mob à débloquer, stockés comme les bits d'un seul entier.

```gdscript
class_name Unlocks
extends RefCounted

const SURVIVED_30S := 1 << 0
const SCORE_100 := 1 << 1
const KILLED_BY_EVERY_TYPE := 1 << 2

var _bits: int = 0

func has(flag: int) -> bool:
	return (_bits & flag) != 0

func grant(flag: int) -> void:
	_bits |= flag
```

**En vrai projet** : justifié pour un ensemble fini et petit. La sérialisation tient en **un seul champ**, donc ajouter un neuvième succès ne casse aucune sauvegarde existante.

**Écho au corpus** : `SamusGear` de [Metroid](./metroid) — les huit capacités du jeu dans un octet, et le mot de passe de débogage qui écrit `$FF` pour tout débloquer. Avec le critère de choix : la relation porte-t-elle un attribut propre ? Non → un bit suffit.

---

## Phase 9 — ce qui ne rentre pas dans ce jeu

Trois familles de concepts du corpus n'ont **rien à faire** dans Dodge the Creeps, et c'est important de le dire plutôt que de les forcer.

| Concept | Pourquoi ça ne rentre pas | Où ça rentrera |
|---|---|---|
| **Fenêtre glissante / streaming** | l'écran est fixe, il n'y a rien à faire défiler ni à décompresser en avance | le donjon grid-based, ou un raycaster |
| **Capteurs et collision directionnelle** | c'est de la physique de plateforme : pentes, boucles, sol traversable par le bas | un platformer, si l'envie vient |
| **Chaîne d'indirections de décor** | il n'y a pas de décor à décrire — un `ColorRect` et c'est tout | le donjon grid-based, où layouts et tuiles partagées reprennent tout leur sens |

Une exception possible si tu veux quand même toucher au streaming : un **fond défilant à tuiles recyclées**, où deux bandes alternent comme les deux `Block_Buffer` de [Mario](./super-mario-bros). C'est honnête et ça se fait en une session — mais c'est un ajout gratuit au jeu, pas une nécessité.

---

## Après Dodge the Creeps — le pont vers le RPG

Quand les phases 0 à 7 seront faites, le RPG tour par tour réutilise **le même outillage** à plus grande échelle. Rien de nouveau à concevoir : la différence n'est pas dans les patterns, mais dans l'échelle et le nombre d'entités à faire coexister.

- **Modèle de données** : `PartyClass` / `PartyMember` reprend `MobType` / `Mob`, lui-même repris de `PokemonSpecies` / `Pokemon`
- **Strategy** pour attaque/sort/objet/défense/fuite, avec la variante de [Final Fantasy](./final-fantasy-1) : séparer `can_execute()` d'`execute()`, ce qui permet de griser un bouton de menu sans dupliquer la règle métier
- **State** pour le déroulement d'un tour, et là c'est le modèle Final Fantasy et non Pokémon qu'il faut prendre : un parti de 4 impose une **file d'acteurs dans l'état**, ordonnée par Agilité, et cette file doit être **validée au moment de sortir un acteur, pas de l'enfiler** (un acteur peut mourir avant son tour)
- **Observer** pour l'UI de combat — quatre barres de PV et des compteurs de charges, c'est le cas où l'appel direct devient intenable
- **Command** pour le journal de combat, offert par-dessus
- **Memento** avec le champ de version déjà rodé à l'étape 7.1

**Trois décisions de modélisation à prendre tôt**, et le corpus donne les critères :

1. **Le grain des ressources de sort** — charges par sort (les PP de Pokémon, 32 compteurs) ou par niveau de sort (les 8 compteurs de Final Fantasy) ? Le second est plus léger et plus lisible, au prix de ne plus pouvoir épuiser un sort en particulier.
2. **Le côté propriétaire des permissions d'équipement** — Final Fantasy les stocke **sur l'objet** (un bit par classe), ce qui rend « puis-je équiper ça ? » gratuit et « que peut porter ce personnage ? » coûteux. Si l'UI affiche une liste filtrée par personnage, il faut l'index inverse, construit une fois au chargement.
3. **La polarité des permissions** — toujours dans le sens positif (« peut »), jamais négatif. Final Fantasy a fait l'inverse, ce qui rend une classe oubliée dans la table **omnipotente** plutôt qu'impuissante.

---

## Le backlog d'analyses

Mega Man, Kirby, Castlevania, un metroidvania plus tardif. Le prompt réutilisable et ses exigences de sourcing sont dans [`_framework/prompt-template.md`](./_framework/prompt-template.md).

Rien n'y est urgent : la roadmap ci-dessus représente environ sept semaines de pratique, et c'est elle qui transforme la théorie en code qui tourne.

Deux pistes techniques identifiées pendant la passe de vérification, mises de côté explicitement :

- **La génération procédurale par fenêtre glissante** — trois jeux du corpus (Mario en colonnes, Final Fantasy en lignes, F-Zero sur les deux axes) utilisent le même principe : décompresser juste en avance de ce qui est consommé, recycler derrière. La différence entre un niveau en ROM et un monde illimité par seed tient entièrement dans l'implémentation de la fonction qui répond à « qu'y a-t-il en (x, y) ? ». [Metroid](./metroid) est le terrain le plus naturel. Deux réserves : garantir la **franchissabilité sous contrainte de capacités** (problème de génération sous contraintes, plus dur que la génération), et garantir le **déterminisme par coordonnée** — sinon le monde se réécrit derrière le joueur.
- **Le verrou codé contre le verrou géométrique** — le corpus contient les deux réponses opposées ([Zelda II](./zelda-2) verrouille en code de quatre façons différentes, [Metroid](./metroid) ne verrouille rien du tout), et le choix est à faire consciemment le jour où un projet aura une progression à bloquer. Un verrou géométrique récompense l'ingéniosité du joueur ; un verrou codé rend le jeu **testable**. Ce qu'il faut éviter, c'est de croire qu'on a un verrou codé alors qu'on n'a qu'un mur.
