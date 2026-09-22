# Design patterns observés — référence transversale

Patterns reconnus au fil des huit décorticages, avec leur idiome Godot et un exemple concret. Document transversal : chaque jeu analysé y renvoie plutôt que de répéter les explications.

## Vue d'ensemble

| Pattern | Rôle | Idiome Godot | Vu dans |
|---|---|---|---|
| Singleton | État global accessible partout | Autoload | Les 8 jeux (niveau 1) |
| Factory | Construire une instance à partir de données | Fonction statique | Pokémon (`from_species`), Mario (expansion d'objet) |
| Prototype | Cloner une instance existante | `.duplicate()` sur une Resource | Pokémon (piège du cache) |
| Composition over inheritance | Référencer plutôt qu'hériter | Resource référencée par une classe runtime | Les 8 jeux (niveau 3) |
| Flyweight | Donnée de référence partagée en mémoire | Cache de Resources par chemin | Les 8 jeux — voir la section dédiée |
| Strategy | Comportement interchangeable | Sous-classes ou `Callable` | Pokémon (effets), Zelda 1 (monstres), Zelda II (sorts), Mario (ennemis), FF1 (actions de combat), Metroid (rayons), Sonic (Badniks), F-Zero (pilotage IA) |
| Observer | Notifier sans connaître les auditeurs | `signal` / `.connect()` | Pokémon (UI de combat), FF1 (4 barres de PV) |
| State | Comportement qui change dans le temps | Objet State imbriqué | Pokémon (tour de combat), Zelda 1 (Link), Zelda II (deux moteurs de mouvement), Mario (formes), FF1 (tour à 4 acteurs), Metroid (boule), Sonic (le cas le plus justifié) |
| Command | Encapsuler une action | Objet exécutable référencé par nom | Zelda II (`Pointer_Table_for_spells_routines`) |
| Object Pool | Réutiliser plutôt qu'instancier/libérer | Array pré-alloué + `reset()` | Mario (block buffers), Sonic (32 anneaux), F-Zero (fenêtre Mode 7) |
| Memento | Sauver un état à l'extérieur de l'objet | `Dictionary` sérialisé en JSON | Zelda 1 (octet de flags par salle) |
| Flag set (bitmask) | Un ensemble fini comme entier | Entier + constantes nommées | Metroid (`SamusGear`), Zelda II (sorts connus) |

Deux remarques avant d'entrer dans le détail.

**Le pattern le plus universel n'est pas dans les livres de GoF au sens où on l'y trouverait décrit pour ce cas.** C'est la **chaîne d'indirections partagées** — cinq des huit jeux décrivent leur décor par deux à quatre niveaux d'indirection, chaque niveau étant lui-même partagé (voir la section Flyweight). Ce n'est pas une astuce par jeu, c'est le format canonique du décor en 8 et 16 bits, et le `TileSet` de Godot en est la version moderne.

**Le pattern le plus mal appliqué dans les originaux est le discriminant.** Quatre jeux du corpus font dépendre le sens d'une valeur d'un contexte extérieur — une plage numérique (Zelda 1 : `< $32` type / `≥ $62` liste ; Metroid : `< $80` solide / `≥ $A0` traversable), une variable d'état globale (Metroid : la banque courante ; Sonic : la zone courante), ou une superposition mémoire (Pokémon : une `UNION`). C'est compact, indolore à l'écriture, et c'est la cause de la moitié des bugs de [`lecons-bugs.md`](./lecons-bugs.md).

## Singleton

Un état accessible depuis n'importe quel point du jeu sans passer de référence de scène en scène. Chez Godot : un autoload. Utilisé dans les huit jeux pour la machine à états globale (niveau 1) — l'overworld notifie l'autoload, qui charge la scène de combat ou de zone suivante.

Nuance apportée par la vérification, et elle compte : **aucun des originaux n'a d'état global unique.** Ce qui tient lieu de machine à états est un ensemble de drapeaux et d'index indépendants — `wIsInBattle`, `wCurMapScript`, `wStatusFlags7` et `wMiscFlags` chez Pokémon, trois choses distinctes posées par trois routines différentes. La moitié des glitches du corpus vient de là : quand l'état est une conjonction de drapeaux, il existe des combinaisons que personne n'a prévues.

Ce que Godot rend gratuit et qu'il faut donc s'imposer :

```gdscript
extends Node                          ## autoload GameState

enum State { EXPLORATION, COMBAT, MENU, DIALOGUE }

signal state_changed(from: State, to: State)

var _state: State = State.EXPLORATION

var state: State:
	get: return _state

## Un seul point d'écriture. Pas de drapeau parallèle, jamais.
func transition_to(next: State) -> void:
	if next == _state:
		return
	var previous := _state
	_state = next
	state_changed.emit(previous, next)
```

## Factory

Une fonction (souvent statique) qui construit une instance complète à partir de données, plutôt que de laisser l'appelant assembler l'objet lui-même.

```gdscript
static func from_species(p_species: PokemonSpecies, p_level: int) -> Pokemon:
	var mon := Pokemon.new()
	mon.species = p_species
	mon.level = p_level
	mon.max_hp = _stat_hp(p_species.base_hp, p_level)
	return mon
```

Variante **un vers plusieurs**, vue chez Mario : un `LevelObject` compact (« plateforme longueur 5 ») produit N cellules de `TileMap`. C'est la forme que prend n'importe quel chargeur de niveau data-driven, et c'est le même geste architectural — une description entre, des objets concrets sortent, l'appelant ne sait pas comment.

## Prototype

Cloner une instance existante plutôt que d'en construire une nouvelle depuis la donnée de référence. Chez Godot : `.duplicate()` sur une Resource, pour le cas où une vraie copie mutable est nécessaire (voir le piège du cache de Resources dans l'analyse Pokémon).

## Composition over inheritance

Pas de hiérarchie de classes par variante (`Bulbizarre extends Pokemon`) : une seule structure de données, qui référence une donnée de configuration plutôt que d'en hériter. Vu dans les huit jeux : `Pokemon` référence un `PokemonSpecies`, un ennemi Zelda référence son `MonsterType`, un bloc Mario référence son `BlockContent`, un `Character` de Final Fantasy référence sa `CharacterClass`, un bloc Sonic référence sa `CollisionShapeData`, un `Racer` de F-Zero référence ses `MachineStats`.

Cas particulier repéré chez Final Fantasy, et il vaut d'être connu : les six classes promues **partagent la courbe de progression** de leur forme d'origine, via une convention numérique (`classe_promue = classe_de_base + 6`). Une jointure implicite réalisée par arithmétique. Élégant, compact, et totalement muet sur son intention — d'où la préférence pour une référence explicite en GDScript :

```gdscript
@export var promoted_form: CharacterClass       ## plutôt que class_id + 6
@export var growth_curve: GrowthCurve           ## référencée, jamais dupliquée
```

## Flyweight

Le nom GoF de ce qu'on utilise sans le nommer depuis le début : une donnée de référence (une Resource chargée depuis un chemin) partagée en mémoire entre toutes les instances qui la référencent, plutôt que dupliquée pour chacune. Chez Godot, c'est automatique via le cache du `ResourceLoader`.

**C'est le seul pattern présent dans les huit jeux sans exception**, et la vérification a montré qu'il y est presque toujours **composé sur plusieurs niveaux** :

| Jeu | Profondeur | Chaîne |
|---|---|---|
| Zelda 1 | 3 | position de carte → layout (121 pour 128) → colonne (150 uniques) → square |
| Mario | 2 | objet de niveau → metatile ; plus une bibliothèque de templates de colonnes |
| Metroid | 3 | salle → structure → macro-bloc (2 × 2 tuiles) → tuile |
| Sonic | 3 | layout → chunk (256 × 256) → bloc (16 × 16) → forme de collision **et** 4 tuiles 8 × 8 |
| F-Zero | 4 | grille 32 × 16 → block (256 px) → band (16 px) → chip (16 × 16) → 2 × 2 tuiles |
| Pokémon | 1 | `Pokemon` → `PokemonSpecies` ; `MoveSlot` → `MoveData` |
| Zelda II | 1 | terrain → profil de rencontre ; sort × niveau → coût |
| Final Fantasy | 1 | `Character` → `CharacterClass` → `GrowthCurve` partagée |

Chaque niveau existe parce qu'il capture **une échelle de répétition différente** — le motif de 16 px, la bande, le quartier. C'est pour ça qu'on ne peut pas les aplatir sans perdre le partage.

Le cas le plus instructif est Sonic, parce qu'il **sépare l'identité graphique de l'identité de collision** : le même numéro de bloc indexe d'un côté une forme de collision partagée, de l'autre quatre tuiles de rendu. Deux blocs visuellement différents peuvent partager exactement la même géométrie, et un bloc peut être visible sans être solide.

**Le piège reste le même partout** : rien n'empêche d'écrire sur la donnée partagée par erreur (voir `target.species.base_defense -= 10` dans l'analyse Pokémon). Une identity map sans dirty-checking, c'est une identity map dont c'est à toi de tenir les invariants.

L'argument à retenir pour Godot, où la mémoire n'est plus la contrainte : **une référence partagée est une garantie d'identité.** Si deux écrans doivent rester visuellement cohérents, les faire pointer vers la même Resource rend la divergence impossible, là où deux copies finiront par diverger à la première retouche.

## Strategy

Une interface commune, plusieurs implémentations interchangeables, choisies par la donnée plutôt que par un `match` qui grossit à chaque ajout.

```gdscript
class_name EnemyBehavior
extends Resource

func update(enemy: Node2D, delta: float) -> void:
	pass

class_name ChaseBehavior
extends EnemyBehavior

func update(enemy: Node2D, delta: float) -> void:
	enemy.position += enemy.direction_to_player() * enemy.speed * delta

class_name FleeBehavior
extends EnemyBehavior

func update(enemy: Node2D, delta: float) -> void:
	enemy.position -= enemy.direction_to_player() * enemy.speed * delta
```

```gdscript
var behavior: EnemyBehavior = ChaseBehavior.new()

func _physics_process(delta: float) -> void:
	behavior.update(self, delta)
	if health < max_health * 0.2:
		behavior = FleeBehavior.new()  ## changement à chaud pendant le jeu
```

Vaut le coup quand plusieurs variantes existent ou vont grossir, et/ou que le comportement doit changer pendant que l'objet vit. Pas la peine pour 2 cas stables qui ne bougeront jamais. Trois niveaux d'engagement chez Godot, du plus au moins formel : classe de base + sous-classes (ci-dessus), duck-typing via `has_method()` sans classe commune, ou un `Callable` stocké dans une variable pour les cas triviaux.

**Le point que les huit jeux ont en commun, et qui est ce qui rend le pattern gratuit** : le comportement est un **champ de la donnée de référence**, pas un branchement dans le moteur. `MoveData.effect`, `MonsterType.behavior`, `EnemyType.behavior`, `BlockDef.collision` sont tous des `@export` de Resource, éditables dans l'inspecteur. Le choix du comportement vit dans le `.tres`, pas dans le code.

**Variante utile, vue chez Final Fantasy** : séparer `can_execute()` d'`execute()`. C'est ce qui permet de griser un bouton de menu sans dupliquer la règle métier.

```gdscript
class_name BattleAction
extends Resource

func can_execute(actor: Character, battle: BattleController) -> bool:
	return true

func execute(actor: Character, targets: Array[Character], battle: BattleController) -> void:
	push_warning("execute() non implémenté")
```

**Variante data-driven poussée, vue chez F-Zero** : la trajectoire est une donnée du *circuit* (254 waypoints, trois profils de pilotage chacun), la façon de la suivre est une donnée du *concurrent*. Séparer les deux permet d'ajouter un adversaire sans toucher aux circuits, et un circuit sans toucher aux adversaires.

## Observer

Le seul de cette liste qui n'est pas à coder à la main chez Godot : les signaux le sont nativement. Le sujet annonce un événement sans savoir qui écoute ni ce qu'ils en font.

```gdscript
class_name Pokemon
extends RefCounted

signal fainted(pokemon: Pokemon)
signal hp_changed(current: int, maximum: int)

func take_damage(amount: int) -> void:
	current_hp = maxi(0, current_hp - amount)
	hp_changed.emit(current_hp, max_hp)
	if current_hp == 0:
		fainted.emit(self)
```

```gdscript
# BattleUI.gd et l'état du combat s'abonnent tous les deux, indépendamment
func _ready() -> void:
	player_pokemon.fainted.connect(_on_fainted, CONNECT_ONE_SHOT)
```

Sans ça, `Pokemon.take_damage()` devrait appeler directement l'UI et la logique de combat — un objet de données qui connaît l'UI, une inversion de couche. Piège à connaître : une connexion qui survit à l'objet qui l'a créée (`disconnect()`, ou `CONNECT_ONE_SHOT` pour un listener à usage unique).

Le cas qui monte en difficulté est Final Fantasy : 4 barres de PV et 8 compteurs de charges à suivre simultanément. Avec un appel direct, l'UI de combat devrait connaître la composition du parti ; avec des signaux, chaque membre annonce et l'UI branche ce qu'elle veut.

## State

À distinguer du niveau 1 (machine à états globale, qui décide QUELLE SCÈNE est active) : ici, le comportement change SANS changer de scène.

```gdscript
class_name BattleState
extends RefCounted

func enter(battle: BattleController) -> void:
	pass

class_name AwaitingInputState
extends BattleState

func handle_input(battle: BattleController, action: String) -> void:
	battle.transition_to(ResolvingState.new(action))

class_name ResolvingState
extends BattleState

var chosen_action: String

func _init(action: String) -> void:
	chosen_action = action

func enter(battle: BattleController) -> void:
	await battle.execute(chosen_action)
	battle.transition_to(CheckVictoryState.new())

class_name CheckVictoryState
extends BattleState

func enter(battle: BattleController) -> void:
	if battle.enemy_fainted():
		battle.end_battle()
	else:
		battle.transition_to(AwaitingInputState.new())
```

Chaque état sait dans quel état aller ensuite : pas de `match current_phase:` géant dans le contrôleur de combat, juste `transition_to(NextState.new())`.

**Trois enseignements tirés de la vérification**, qui n'étaient pas dans la version précédente de ce document.

**1. Un état qui porte une file doit valider au moment de sortir, pas d'enfiler.** Chez Final Fantasy, l'ordre des tours est calculé en début de round par Agilité, mais un acteur peut mourir avant son tour :

```gdscript
func enter(battle: BattleController) -> void:
	var actor: Character = battle.turn_queue.pop_front()
	if not actor.is_alive():                    ## indispensable
		battle.transition_to(NextActorState.new())
		return
```

**2. Le cas où State devient obligatoire, c'est quand l'état change une dimension.** Sonic est l'exemple : accélération, friction, décélération **et les rayons de collision** changent entre debout et roulade. Un booléen `is_rolling` devrait être testé à sept endroits.

**3. Et il faut alors dériver les dimensions depuis l'état, pas les y copier.** C'est le bug documenté de Sonic 1 : les capteurs de poussée ne sont pas repositionnés en roulade. Copier crée une seconde source de vérité.

```gdscript
## Fragile : le changement d'état doit penser à tout mettre à jour.
func set_state(s: SonicState) -> void:
	_state = s
	width_radius = s.width_radius()             ## et si on en oublie un ?

## Robuste : rien à synchroniser, tout dérive de l'état courant.
func width_radius() -> float:
	return _state.width_radius()
```

## Command

Encapsuler une action dans un objet plutôt que d'exécuter du code directement au moment de l'input — utile pour le remapping de touches, et gratuit pour l'undo/replay (un historique d'actions pour un log de combat, par exemple).

```gdscript
class_name InputCommand
extends RefCounted

func execute(player: Node) -> void:
	pass

class_name JumpCommand
extends InputCommand

func execute(player: Node) -> void:
	player.jump()
```

```gdscript
var bindings: Dictionary = {
	"jump_button": JumpCommand.new(),
	"attack_button": AttackCommand.new(),
}

func handle_input(button: String) -> void:
	if bindings.has(button):
		bindings[button].execute(player)
```

Remapper une touche = échanger l'entrée du Dictionary, pas modifier un `if/elif` dispersé dans le code.

**Zelda II le fait déjà, sans le nommer** : le désassemblage contient une `Pointer_Table_for_spells_routines` de huit entrées, une routine par sort, invoquée par index. La transposition en tire le bénéfice secondaire du pattern — l'historique :

```gdscript
func cast(spell: SpellCostTable.Spell, link: Node2D) -> bool:
	if not LinkProgress.knows(spell):
		return false
	_effects[spell].cast(link)
	_cast_log.append(spell)                     ## journal de combat gratuit
	return true
```

Pour un RPG au tour par tour, c'est exactement ce qui alimente un journal de combat sans code dédié.

## Object Pool

Réutiliser des instances plutôt qu'instancier/libérer en boucle — pertinent pour des projectiles, particules, ou tout objet créé/détruit à haute fréquence.

```gdscript
class_name BulletPool
extends Node

var _pool: Array[Bullet] = []

func get_bullet() -> Bullet:
	for b in _pool:
		if not b.visible:
			b.reset()
			return b
	var new_bullet := Bullet.new()
	_pool.append(new_bullet)
	add_child(new_bullet)
	return new_bullet

func release(b: Bullet) -> void:
	b.visible = false
	b.set_physics_process(false)
```

À réserver aux cas où l'instanciation répétée est un problème mesuré (profiler avant d'optimiser) — sur un projet à l'échelle d'un hobby, `instantiate()`/`queue_free()` suffit dans l'immense majorité des cas.

**Le critère précis, que la vérification permet maintenant d'énoncer** : pas « beaucoup d'objets » mais « beaucoup d'objets **créés et détruits en rafale** ». Le cas de référence du corpus est Sonic : **32 anneaux dans la même frame**, détruits quelques secondes plus tard, à répétition pendant toute la partie. L'original les limite d'ailleurs par le nombre de créneaux d'objets libres, pas par un choix de design. À l'inverse, les mobs de Dodge the Creeps ne remplissent pas le critère (voir [`../prochaines-etapes.md`](../prochaines-etapes.md)).

**Le pool est aussi le bon endroit pour distribuer la charge.** Les anneaux dispersés de Sonic ne testent le sol qu'**une frame sur quatre**, avec un décalage de phase par anneau. Pré-allouer et échelonner sont deux problèmes distincts qui se résolvent au même endroit :

```gdscript
func _ready() -> void:
	for i in SIZE:
		var r := ScatteredRing.new()
		r.ground_check_phase = i % 4            ## échelonnement de la charge
		add_child(r)
		_pool.append(r)
```

### La généralisation : la fenêtre glissante

Trois jeux du corpus généralisent le pool en **fenêtre glissante** sur le décor, et chacun sur un axe différent :

| Jeu | Fenêtre | Axe |
|---|---|---|
| Mario | 2 pages de 16 × 13 metatiles (`Block_Buffer_1/2`) | colonnes |
| Final Fantasy | 16 lignes de 256 tuiles en RAM | lignes |
| F-Zero | 1024 × 1024 px de tilemap Mode 7 | les deux |

Le principe est le même dans les trois cas : **décompresser juste en avance de ce qui est consommé, recycler derrière.** Et il ne dépend pas de la contrainte mémoire qui l'a fait naître — c'est la base de n'importe quel monde théoriquement sans limite. La différence entre « niveau 1-1 en ROM » et « monde infini par seed » tient entièrement dans l'implémentation de la fonction qui répond à « qu'y a-t-il en (x, y) ? ».

Détail développé dans [Mario](../super-mario-bros) (niveau 4) et [Metroid](../metroid) (niveau 4), avec les deux réserves qui comptent pour un metroidvania procédural : garantir la **franchissabilité sous contrainte de capacités**, et garantir le **déterminisme par coordonnée** (sinon le monde se réécrit derrière le joueur).

## Memento

Sauver l'état d'un objet à l'extérieur de cet objet, sous une forme opaque qu'il sait relire, sans exposer sa structure interne. Zelda 1 en est la version minimale : **un octet par salle, 128 octets par monde**.

```gdscript
class_name RoomFlags
extends RefCounted

var doors_opened: int = 0          ## bitmask 4 directions
var item_taken: bool = false
var visited: bool = false
var kill_counter: int = 0          ## saturé, pas exact
var secret_found: bool = false

func to_dict() -> Dictionary:
	return {
		"doors": doors_opened,
		"item": item_taken,
		"visited": visited,
		"kills": kill_counter,
		"secret": secret_found,
	}
```

**La leçon n'est pas technique, elle est de granularité.** Zelda 1 ne stocke qu'un compteur de monstres tués **saturé** (2 bits en donjon, 3 en overworld), jamais leur identité ni leur position. Le réflexe naturel sur PC serait de sérialiser chaque ennemi individuellement — et c'est le piège : ça transforme une sauvegarde de quelques centaines d'octets en sérialisation d'un monde entier, et ça oblige à décider ce qui se passe quand le peuplement d'une salle change entre deux versions du jeu (une sauvegarde qui référence un ennemi supprimé, c'est une clé étrangère cassée).

Attention aussi à la **symétrie du couple `to_dict()` / `from_dict()`**. Le bug TMPR/SABR de Final Fantasy est exactement ça : l'effet est calculé correctement, puis jeté parce que les deux routines de sérialisation ne couvrent pas le même ensemble de champs. Tout compile, la valeur disparaît silencieusement.

## Flag set (bitmask)

Un ensemble fini, petit et connu à la compilation, stocké comme un entier. Metroid en est le cas pur : **les huit capacités du jeu tiennent dans un seul octet** (`SamusGear`), et le mot de passe de débogage écrit littéralement `$FF` pour tout débloquer.

```gdscript
class_name AbilitySet
extends RefCounted

const BOMBS := 1 << 0
const HIGH_JUMP := 1 << 1
const MORPH_BALL := 1 << 4
const VARIA := 1 << 5
const ICE_BEAM := 1 << 7

const NAMES := {
	&"bombs": BOMBS,
	&"high_jump": HIGH_JUMP,
	&"morph_ball": MORPH_BALL,
	&"varia": VARIA,
	&"ice_beam": ICE_BEAM,
}

var _bits: int = 0

func has(ability: StringName) -> bool:
	return (_bits & NAMES.get(ability, 0)) != 0

func grant(ability: StringName) -> void:
	_bits |= NAMES.get(ability, 0)
```

**Le critère de choix face à une entité de jointure** est net, et c'est le même qu'en modélisation relationnelle : **la relation porte-t-elle un attribut propre ?** Le movepool de Pokémon porte des PP par capacité, donc il faut une entité (`MoveSlot`). Les capacités de Metroid ne portent rien — une fois obtenues, elles sont acquises — donc un bit suffit.

Deux avantages pratiques au bitmask, au-delà de la compacité : la sérialisation tient en **un seul champ** (ajouter une neuvième capacité ne casse aucune sauvegarde existante), et un `@export_flags` rend l'ensemble éditable dans l'inspecteur sans code.

**Le piège, vu chez Final Fantasy** : la polarité. Sa LUT de permission magique utilise « bit mis = **ne peut pas** lancer », ce qui rend la donnée par défaut (tout à zéro) maximalement **permissive**. Toute classe oubliée dans la table serait omnipotente. Aligner le défaut du langage — zéro, vide, `false` — sur le comportement le plus restrictif est gratuit et évite toute une classe d'erreurs.
