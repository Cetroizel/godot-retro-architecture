# Prochaines étapes — de l'analyse à la pratique

Huit jeux décortiqués, un référentiel de patterns, une référence de bugs, un glossaire, et une passe de vérification contre les désassemblages. Cette phase d'analyse se referme ici — la suite se joue dans Godot, pas dans de nouveaux fichiers Markdown.

## Étape 1 — Dodge the Creeps comme POC volontaire

Les modifications déjà prévues sur ce projet ne changent pas, mais chacune devient l'occasion d'appliquer un pattern précis plutôt que la première solution qui marche :

| Modification prévue | Pattern à appliquer | Pourquoi |
|---|---|---|
| Nouveau type de mob à comportement différent | **Strategy** | Exactement l'exemple `EnemyBehavior`/`ChaseBehavior`/`FleeBehavior` du référentiel — un nouveau comportement = une nouvelle sous-classe, zéro `match` qui grossit |
| Difficulté progressive | **Flyweight** | Une `WaveData` Resource par palier de difficulté (fréquence de spawn, vitesse des mobs), référencée plutôt que codée en dur dans le spawner |
| Power-up temporaire | **State** | Le joueur bascule entre un état Normal et un état PoweredUp avec sa propre durée et ses propres règles, plutôt qu'un booléen `has_powerup` vérifié partout |
| (déjà en place) Score | **Observer** | Remplacer l'appel direct à l'UI par un signal `score_changed` — le score ne devrait pas savoir que l'UI existe |
| Nouveau sprite joueur/mob | — | Pas de pattern à appliquer, travail d'asset pur |

**Object Pool reste volontairement absent de cette liste**, et la vérification a rendu le critère plus précis qu'un simple « trop peu de mobs ». Le critère n'est pas *beaucoup d'objets* mais **beaucoup d'objets créés et détruits en rafale** : le cas de référence du corpus est Sonic, avec ses **32 anneaux dispersés dans la même frame** — et l'original les limite d'ailleurs par le nombre de créneaux d'objets libres, pas par un choix de design. Dodge the Creeps spawne ses mobs un par un, à intervalle régulier : le coût d'`instantiate()`/`queue_free()` n'y sera jamais mesurable. Un projet avec un vrai pic de créations simultanées (des projectiles, des particules, le donjon grid-based du backlog) sera le bon moment.

Ordre suggéré : Strategy d'abord (le plus direct, un seul fichier à ajouter par comportement), puis Observer sur le score (mécanique déjà en place, juste à découpler), puis State pour le power-up, puis Flyweight pour la difficulté progressive une fois qu'il y a plusieurs types de mobs à faire varier ensemble.

### Deux précisions tirées de la vérification, à appliquer dès ce projet

**Sur le State du power-up** — Sonic a livré la raison exacte pour laquelle ce pattern est obligatoire plutôt que cosmétique : un état qui change une **dimension** (un rayon de collision, une vitesse maximale, un plafond) doit voir cette dimension **dérivée depuis l'état**, jamais copiée vers l'objet. Le bug documenté de Sonic 1 est précisément là — les capteurs de poussée ne sont pas repositionnés en roulade, alors que les rayons de collision changent. Concrètement, pour le power-up :

```gdscript
## Fragile : chaque changement d'état doit penser à tout mettre à jour.
func set_powered_up(active: bool) -> void:
	speed = 700 if active else 400
	## ... et l'invincibilité, et le sprite, et la hitbox ?

## Robuste : rien à synchroniser.
func current_speed() -> float:
	return _state.speed()
```

**Sur l'Observer du score** — le piège à connaître est la connexion qui survit à l'objet qui l'a créée. Un `CONNECT_ONE_SHOT` pour ce qui ne doit arriver qu'une fois (la mort du joueur), un `disconnect()` explicite sinon.

## Étape 2 — le pont vers le RPG tour par tour

Quand Dodge the Creeps aura servi de validation, le projet RPG réutilise directement le même outillage, à une échelle plus grande :

- **Modèle de données** : `PartySpecies`/`PartyMember` reprend `PokemonSpecies`/`Pokemon` presque tel quel — classe de personnage en Resource partagée, instance de personnage qui référence sa classe
- **Strategy** pour attaque/sort/objet/défense/fuite, comme les effets de capacité Pokémon — avec la variante de [Final Fantasy](./final-fantasy-1) : séparer `can_execute()` d'`execute()`, ce qui permet de griser un bouton de menu sans dupliquer la règle métier
- **State** pour le déroulement d'un tour, et là c'est le modèle Final Fantasy et non Pokémon qu'il faut prendre : un parti de 4 impose une **file d'acteurs dans l'état**, ordonnée par Agilité, et cette file doit être **validée au moment de sortir un acteur, pas de l'enfiler** (un acteur peut mourir avant son tour)
- **Observer** pour que l'UI de combat réagisse aux PV et à la mort sans coupler l'affichage à la logique métier — 4 barres de PV et des compteurs de charges, c'est le cas où l'appel direct devient intenable
- **Command** pour le journal de combat, offert par-dessus : [Zelda II](./zelda-2) le fait déjà avec sa table de pointeurs de sorts, et encapsuler une action dans un objet rend l'historique et le replay gratuits

Rien de nouveau à concevoir : la différence entre Dodge the Creeps et le RPG n'est pas dans les patterns utilisés, mais dans l'échelle et le nombre d'entités à faire coexister.

**Trois décisions de modélisation à prendre tôt**, et le corpus donne les critères :

1. **Le grain des ressources de sort.** Des charges par sort (les PP de Pokémon, 32 compteurs) ou par niveau de sort (les 8 compteurs de Final Fantasy) ? Le second est plus léger et plus lisible, au prix de ne plus pouvoir épuiser un sort en particulier. Pour un jeu au ton humoristique où la gestion fine n'est pas le sujet, le grain grossier est probablement le bon.
2. **Le côté propriétaire des permissions d'équipement.** Final Fantasy les stocke **sur l'objet** (un bit par classe), ce qui rend « puis-je équiper ça ? » gratuit et « que peut porter ce personnage ? » coûteux. Si l'UI affiche une liste filtrée par personnage, il faut l'index inverse — construit une fois au chargement.
3. **La polarité des permissions.** Toujours dans le sens positif (« peut »), jamais négatif (« ne peut pas »). Final Fantasy a fait l'inverse, ce qui rend une classe oubliée dans la table **omnipotente** plutôt qu'impuissante.

## Ce qui reste ouvert

Le backlog de jeux à décortiquer (Mega Man, Kirby, Castlevania, un metroidvania plus tardif) reste disponible si l'envie de reprendre l'analyse revient — mais rien n'y est urgent tant que ces deux étapes de pratique n'ont pas transformé la théorie en code qui tourne.

Deux pistes techniques identifiées pendant la vérification, mises de côté explicitement :

- **La génération procédurale par fenêtre glissante** — trois jeux du corpus (Mario en colonnes, Final Fantasy en lignes, F-Zero sur les deux axes) utilisent le même principe : décompresser juste en avance de ce qui est consommé, recycler derrière. La différence entre un niveau en ROM et un monde illimité par seed tient entièrement dans l'implémentation de la fonction qui répond à « qu'y a-t-il en (x, y) ? ». [Metroid](./metroid) est le terrain le plus naturel — monde adressé par coordonnées, salles résolues à la demande, cases vides déjà gérées comme un cas normal. Les deux réserves à ne pas oublier : garantir la **franchissabilité sous contrainte de capacités** (un problème de génération sous contraintes, plus dur que la génération), et garantir le **déterminisme par coordonnée** (sinon le monde se réécrit derrière le joueur — même règle que la seed FNV-1a du desktop pet).
- **Le verrou codé contre le verrou géométrique** — le corpus contient les deux réponses opposées ([Zelda II](./zelda-2) verrouille en code de quatre façons différentes, [Metroid](./metroid) ne verrouille rien du tout), et le choix est à faire consciemment le jour où un projet aura une progression à bloquer. Un verrou géométrique récompense l'ingéniosité du joueur ; un verrou codé rend le jeu **testable**. Ce qu'il faut éviter, c'est de croire qu'on a un verrou codé alors qu'on n'a qu'un mur.
