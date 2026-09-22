# Leçons de bugs — référence transversale

Sept glitches rencontrés au fil des huit décorticages, répartis en **six familles**. Document transversal, comme [`design-patterns.md`](./design-patterns.md) — chaque jeu y renvoie plutôt que de répéter l'explication complète.

La version précédente de ce document ramenait tout à une seule cause profonde : *« une donnée ou un état utilisé sans être sûr qu'il a été correctement mis en place pour le contexte courant »*. La vérification a montré que c'est vrai pour quatre familles sur six, et **faux pour les deux dernières** — celles où la donnée est parfaitement valide et où c'est le code ou l'état qui est en tort. La classification en six familles est plus utile, parce que chaque famille a une parade différente.

## Vue d'ensemble

| Jeu | Bug | Famille | Mécanisme en une ligne | Parade |
|---|---|---|---|---|
| [Pokémon](../pokemon-rouge-bleu) | MissingNo / Old Man | Lecture non réinitialisée — **temps** | un emplacement à double usage, dont l'invalidation est conditionnée à la donnée entrante | invalidation inconditionnelle, ou pas de double usage |
| [Pokémon](../pokemon-rouge-bleu) | Trainer-Fly / Mew | État en drapeaux épars — **conjonction** | trois drapeaux posés par trois routines, une seule les remet à zéro | un seul champ d'état, un seul point d'écriture |
| [Mario](../super-mario-bros) | Minus World | Donnée jamais initialisée — **chemin** | le seul chemin qui initialise a été contourné | sentinelle bruyante, pas de valeur de remplissage plausible |
| [Zelda II](../zelda-2) | Healer glitch / Scroll Lock | Index hors plage — **plage** | `$69BA,x` avec X = `$0A` écrase la limite de scroll | `clampi()` et assertion sur la taille du tableau |
| [Metroid](../metroid) | Secret Worlds (Door Glitch) | Mauvais contexte — **résolution** | index valide, résolu dans la table d'une autre région | référence directe plutôt qu'index relatif |
| [Final Fantasy](../final-fantasy-1) | Taux de critique, élément d'attaque, TMPR | Mix-up de champ — **code** | le mauvais champ lu, une fois pour toutes, à l'écriture | nommage distinctif, test de bout en bout |
| [Sonic](../sonic) | Capteur de poussée en roulade | État partiellement appliqué — **état** | un changement d'état met à jour certaines dimensions, pas toutes | dériver depuis l'état, ne pas y copier |
| [F-Zero](../fzero) | *aucun* | — | presque aucun état mutable à corrompre | — |

Les quatre premières familles concernent une **donnée invalide lue**. Les deux dernières concernent une donnée parfaitement valide, mal utilisée. C'est la distinction qui décide de la parade : les quatre premières se traitent par des gardes à la lecture, les deux dernières par de la conception.

## 1. Lecture non réinitialisée — Pokémon (MissingNo)

Un emplacement mémoire sert deux usages selon le moment. Le tutoriel du Vieil Homme copie le nom du joueur dans la zone des données de rencontre en herbe, pour afficher « OLD MAN » — l'adresse concernée, `$D887`, est précisément `wGrassRate`, **l'octet de taux** de rencontre, et les onze octets du nom débordent sur les cinq premiers créneaux de la table.

**Le point que la version précédente de ce document manquait** : l'invalidation existe. Elle est simplement **conditionnée à la donnée entrante** —

```
ld [wGrassRate], a
and a
jr z, .NoGrassData        ; taux à 0 → on ne recopie rien
```

Une carte à taux « herbe » 0 ne réinitialise donc rien. Et un second bug s'y ajoute : le jeu décide **s'il y a** rencontre d'après une tuile, et **quelle table lire** d'après une autre. Sur les demi-blocs de littoral, les deux divergent — rencontre au taux de l'eau, données lues dans la table de l'herbe.

**Les trois parades, par ordre de solidité** :

1. Ne pas avoir de double usage. Deux champs séparés coûtent quelques octets qu'on a largement.
2. Invalidation **inconditionnelle** à l'entrée de contexte. Un `_reset()` qui s'exécute toujours, jamais sous condition de la donnée qui arrive.
3. Décider *si* et décider *quoi* à partir de **la même source**. Un seul appel, une seule lecture, une seule décision.

## 2. État en drapeaux épars — Pokémon (Trainer-Fly)

Interrompre par un Vol la séquence déclenchée quand un dresseur repère le joueur laisse le jeu incohérent. **Il n'y a pas de drapeau « transition de combat en cours »** — la version précédente de ce document le supposait. Trois choses indépendantes sont posées :

- `wStatusFlags7` bit « trainer wants to battle » ;
- `wCurMapScript` incrémenté pour que l'étape suivante du script de carte lance le combat ;
- `wMiscFlags` bit « vu par un dresseur », posé plus tôt par la routine de détection.

Seules les routines de début et de fin de combat les remettent à zéro, et le Vol les contourne. Le menu qui ne répond plus n'est pas « désactivé » : un test du bit « vu par un dresseur » **saute l'appel qui l'affiche**.

La suite du glitch (Mew) repose sur une **`UNION` mémoire** : les deux octets lus pour l'espèce et le niveau recouvrent des variables de combat — la statistique Spéciale non modifiée et le cran d'Attaque du dernier adversaire. D'où l'espèce `$15` = 21 = Mew, niveau 7. Deux schémas de données superposés sur les mêmes octets, avec un discriminant implicite que rien ne vérifie à la lecture. L'équivalent moderne le plus proche est une table à colonnes polyvalentes (`value_int`, `value_ref`, dont le sens dépend d'une colonne `type`).

**La parade est architecturale, et Godot la rend gratuite** : un `enum` d'état, **un seul** champ qui le porte, **un seul** point d'écriture. Quand l'état est une valeur unique, il n'existe pas de combinaison imprévue.

```gdscript
var _state: State = State.EXPLORATION

func transition_to(next: State) -> void:
	if next == _state:
		return
	var previous := _state
	_state = next
	state_changed.emit(previous, next)
```

## 3. Donnée jamais initialisée — Mario (Minus World)

Le chemin d'entrée normal dans la Warp Zone est le seul qui affecte la destination des tuyaux. Le clip de mur permet d'arriver dans la même pièce sans passer par ce chemin. TASVideos le formule sans détour : *« it's just an oversight of the programmers that you can get into the pipe before the correct warp labels are assigned »*.

**Et la table responsable porte son propre aveu dans le désassemblage** :

```
WarpZoneNumbers: .db $04, $03, $02, $00 ;warp zone numbers, note spaces on middle
                 .db $24, $05, $24, $00 ;zone, partly responsible for
                 .db $08, $07, $06, $00 ;the minus world
```

La rangée du milieu est celle d'une Warp Zone à une seule destination. Les deux emplacements inutilisés sont remplis avec `$24` = 36, choisi comme valeur « espace ». **Et c'est cette même valeur qui sert de numéro de monde.** L'affichage « -1 » n'est ni une dizaine vide ni un chiffre invalide : le numéro de monde est dessiné avec une tuile d'indice égal au numéro, et `$24` est la tuile **espace** du jeu de tuiles. « 36-1 » s'affiche « ␠-1 ».

Différence avec MissingNo : il n'y a même pas d'ancien usage légitime de cet emplacement à ce moment précis, juste une init qui ne s'est jamais produite.

**Deux parades, dont une qui n'était pas dans la version précédente** :

1. **Une valeur de remplissage n'est pas une valeur neutre.** `$24` a été choisi pour une propriété *de rendu* (il s'affiche blanc), et a fini utilisé comme *numéro de monde*, où il n'a aucun sens. Un remplissage qui traverse une frontière de couche devient une donnée réelle. Une sentinelle doit **planter bruyamment** — `-1`, `null`, une assertion — jamais produire un résultat plausible.
2. Vérifier l'état, pas le chemin. Si la destination doit être affectée avant d'entrer dans le tuyau, c'est l'entrée du tuyau qui doit l'exiger (`assert(destination != null)`), pas le chemin d'arrivée qui doit la garantir.

## 4. Index hors plage — Zelda II (healer glitch, scroll lock)

**C'est la correction la plus lourde de ce document.** La version précédente présentait « Glitch Town » comme *« un vrai garde-fou, qui a quand même ses propres angles morts »* — un écran de secours vers lequel le jeu se redirigerait volontairement plutôt que de planter, et en tirait une leçon sur les chemins de repli mal testés.

**C'est faux. Le code ne contient aucune vérification de borne, nulle part** :

- `$0748` indexe quatre tables parallèles de **63 entrées** — aucun test de plage ;
- le code de ville et le code de palais sont calculés **par soustraction, en aveugle** (`($0748 - $2C) / 2` et `$0748 - $34`), donc un index hors plage produit une soustraction qui boucle et un index arbitraire ;
- le numéro de « world » est calculé sur **0 à 7** alors que la table de pointeurs correspondante ne contient que **6 entrées**.

Aucune source consultable ne documente de fallback intentionnel, et le nom « Glitch Town » lui-même n'a pas pu être retrouvé. Il n'y a pas de filet, donc pas de leçon sur les filets mal testés.

**Le bug réellement documenté est un débordement de tableau.** Une routine met en cache les données de scroll d'une salle dans une série de tableaux indexés par X — `$697B,x`, `$6982,x`, … `$69BA,x`. Or `$69C4` est `STOP_SCROLLING_LEFT_AT_THIS_MAP_PAGE`, soit exactement `$69BA + $0A`. Quand X vaut `$0A`, l'écriture **écrase la limite de scroll gauche** et verrouille le défilement définitivement. Le commentaire du désassemblage, à cet endroit :

> *« This is where healer glitch writes 69C4 at 1, triggering scroll lock — X=A »*

**Et le plus instructif est interne au jeu** : l'index de rencontre en vue de côté, lui, **est correctement borné**. Deux tests amont et la règle de franchissement garantissent que seuls les terrains `$04`–`$0A` peuvent déclencher une rencontre — exactement les sept entrées de la table. Ce n'est pas une équipe qui ignorait la question du bornage. C'est une équipe qui l'a traitée là où elle y a pensé.

**Les parades, et c'est la famille la plus mécanique à éliminer** :

1. Le bornage se raisonne **par frontière de donnée**, pas par réputation de la base de code. Avoir validé une entrée quelque part ne dit rien des autres.
2. `clampi()` coûte une ligne. Une assertion de développement sur la taille du tableau en coûte une autre.

```gdscript
func room_at(index: int) -> RoomData:
	assert(index >= 0 and index < _rooms.size(), "index de salle hors plage : %d" % index)
	return _rooms[clampi(index, 0, _rooms.size() - 1)]
```

## 5. Mauvais contexte de résolution — Metroid (Secret Worlds)

Le nom établi est **Secret Worlds** (ou Hidden Worlds), déclenché par le **Door Glitch** — rester dans une porte bleue et enchaîner mise en boule et démorphage-saut pour traverser le mur.

Le mécanisme découle directement de la structure du jeu. Zebes est **une grille globale unique de 32 × 32 écrans** (et non un graphe, contrairement à ce qu'annonçait la version initiale de l'analyse), les régions en sont des **sous-zones disjointes**, le numéro de salle lu dans une case est **résolu dans les tables de la banque de ROM courante**, et l'ascenseur — le seul mécanisme qui change de banque — **ne modifie pas les coordonnées**.

Donc : sortir de sa sous-zone sans passer par un ascenseur fait continuer les coordonnées, renvoyer des numéros de salle valides, et les résoudre dans les mauvaises tables. Les salles sont reconstruites avec les mauvaises structures et les mauvais ennemis.

**Ce qui distingue cette famille des trois précédentes, et c'est ce qui en fait la plus difficile** : la donnée lue est **parfaitement valide**. Le numéro de salle existe, la case de grille existe, la table existe, la lecture réussit. Ce qui est faux, c'est l'**appariement** entre l'index et la table — et rien, dans la donnée elle-même, ne permet de le détecter. Aucun garde à la lecture n'attrape ça.

En vocabulaire relationnel : une clé étrangère **sans contrainte**, dont la table cible dépend d'une variable de session.

**La parade est de conception : un identifiant ne doit pas dépendre d'un contexte global pour avoir un sens.**

```gdscript
## Fragile : le sens du numéro dépend d'une variable d'état ailleurs.
var room_number: int
func resolve() -> RoomData:
	return GameState.current_region.rooms[room_number]

## Robuste : la référence porte son propre contexte.
@export var room: RoomData
```

Le second n'est pas seulement plus lisible : il est **invérifiable-par-erreur**. Une référence de Resource cassée se voit au chargement ; un index résolu dans la mauvaise table ne se voit jamais.

Trois jeux du corpus font le même choix fragile — le numéro de salle de Metroid résolu par banque, le numéro de bloc de Sonic résolu par zone, le code de ville de Zelda II calculé depuis un index de zone. Et l'aliasing de la tilemap Mode 7 de [F-Zero](../fzero) en est la version latente, rendue inatteignable par le design.

## 6. Mix-up de champ — Final Fantasy

Le mauvais champ lu, une fois pour toutes, à l'écriture du code. Aucune histoire de mémoire ni de timing : le bug est présent et identique à **chaque exécution**, y compris dans des conditions parfaitement normales. La version précédente en retenait un exemplaire ; il y en a trois, et le meilleur n'était pas celui-là.

**Le plus spectaculaire : le taux de critique est l'index d'inventaire de l'arme.**

```
LDY #btlch_critrate
AND #$7F
STA (btl_ib_charstat_ptr), Y   ; BUGGED - this sets the critical rate to the weapon index,
                               ;  rather than actually fetching the critical rate from the weapon stats.
```

L'octet « taux de critique » des données d'arme est de la **donnée morte**. Le taux de critique d'un personnage est proportionnel à **la position de son arme dans la liste d'objets** — d'où le fait que Masamune, dernière de la liste, ait le meilleur taux du jeu, par accident pur. Une seule instruction manquante sépare le jeu de son comportement voulu.

**Le doublet de l'élément d'attaque.** Côté ennemi, le code lit l'élément de **faiblesse** de l'attaquant et l'écrit comme élément **de son attaque** (d'où un loup de glace qui attaque au feu). Côté joueur, la même erreur sur l'élément **et** sur la catégorie de l'arme — ce qui explique que **les épées élémentaires de la version NES ne fassent rien**.

**Le troisième, d'une autre nature : le buff jeté à la frontière.** TMPR et SABR calculent correctement leur effet, mais les routines qui sérialisent les stats d'un joueur ciblé **ne transportent pas le champ concerné**. Le buff est calculé, puis jeté. La formule est juste ; c'est le couple charger/sauver qui est asymétrique — l'équivalent exact d'un DTO auquel il manque un champ.

Mention pour compléter : la statistique **Intelligence est morte**. Deux occurrences dans tout le jeu, aucune lecture par le moteur. Du code de plomberie écrit, appelé, et dont le résultat n'est branché sur rien.

**Les parades** :

1. **Le nommage est le seul garde-fou réaliste.** `weapon_inventory_index` et `weapon_critical_rate` ne se confondent pas ; `index` et `rate` si. Deux entiers du même type ne produisent aucun avertissement de compilation.
2. **Une donnée jamais lue est un bug silencieux, pas une donnée inutile.** Un test qui vérifie qu'un champ influence bien une sortie observable attrape ça ; la relecture de code, non.
3. **Une frontière de sérialisation est une surface à couvrir autant que la logique.** À chaque `to_dict()`, se demander ce qui manque dans le `from_dict()`.

**Et le point opérationnel** : ce sont les **seuls bugs du corpus qu'un test unitaire aurait trouvés.** Les quatre autres familles exigent des conditions particulières — une carte précise, une manipulation, un débordement, une sortie de région. Celles-ci sont présentes à chaque exécution, sur le chemin nominal.

## 7. État partiellement appliqué — Sonic (capteur de poussée en roulade)

En roulade, les rayons de collision de Sonic changent : la largeur passe de 9 à 7, la hauteur de 19 à 14. Les capteurs de sol et de plafond suivent, puisqu'ils sont positionnés depuis ces rayons. **Les capteurs de poussée, eux, ne bougent pas.** Le correctif n'existe que dans les branches `FixBugs` du désassemblage — identifié par la communauté, laissé optionnel pour préserver le comportement d'origine.

Conséquence : en roulade, Sonic est poussé par les murs comme s'il avait encore sa largeur debout. La géométrie de collision n'est pas cohérente avec elle-même.

**C'est la seule famille du corpus qui ne concerne aucune donnée invalide.** Rien n'est corrompu, tout est cohérent en mémoire, aucune lecture ne se fait hors contexte. Un sous-ensemble des conséquences d'un changement d'état a été appliqué, et le reste est resté sur les valeurs précédentes.

**La parade : dériver les dimensions depuis l'état, jamais les y copier.**

```gdscript
## Fragile : le changement d'état doit penser à tout mettre à jour.
func set_state(s: SonicState) -> void:
	_state = s
	width_radius = s.width_radius()
	height_radius = s.height_radius()
	## ... et le capteur de poussée, si on y pense.

## Robuste : rien à synchroniser, tout dérive de l'état courant.
func sensor_offset(sensor: SensorSet.Sensor) -> Vector2:
	sensors.width_radius = _state.width_radius()
	sensors.push_radius = _state.push_radius()
	return sensors.offset_of(sensor, ground_angle)
```

Copier les valeurs de l'état vers le personnage crée une **seconde source de vérité** — et on retombe exactement sur le problème des deux variables de direction de Zelda II. Dériver à la lecture coûte quelques multiplications par frame et rend l'oubli impossible.

C'est aussi, de tout le corpus, **le bug le plus facile à écrire soi-même** : il ne demande aucune contrainte matérielle, aucune économie de mémoire, aucune astuce de 1986. Juste un état avec plus de conséquences qu'on n'en a listé.

## 8. Le cas F-Zero : pas de glitch, et c'est une conclusion

Aucun glitch F-Zero solidement documenté n'illustre une leçon d'architecture. Ce qui a été écarté, et pourquoi : deux circuits du mode Practice sont atteignables en forçant l'index en RAM au-delà de sa plage — mais c'est du **contenu coupé** rendu accessible par triche, pas un bug atteignable en jouant ; l'aliasing de la tilemap Mode 7 est bien une lecture hors contexte, mais **invisible en version commerciale**, la fenêtre streamée étant toujours plus large que le champ de vision.

**Et l'absence est instructive.** F-Zero est le jeu du corpus qui manipule le moins de données mutables : pas d'inventaire, pas de progression sauvegardée, pas de monde à explorer, presque tout en lecture seule. Or les cinq premières familles ci-dessus **supposent toutes** un état mutable ou une table résolue dynamiquement.

Le corollaire est directement utilisable : **la quantité d'état mutable d'un système est une bonne estimation de sa surface de bugs.** Un jeu de course n'a pas moins de bugs qu'un RPG parce qu'il est mieux écrit, mais parce qu'il a moins à se souvenir. Réduire l'état mutable est une stratégie de correction avant d'être une stratégie de performance.

## Check-list pour tes propres projets

Six questions, une par famille — dans l'ordre où elles coûtent le moins cher à vérifier.

**Index et plages** (le plus mécanique)

- Chaque lecture indexée est-elle bornée à la frontière où la donnée entre ? `clampi()` plus une assertion de développement, systématiquement. Le fait d'avoir validé ailleurs ne compte pas.

**Identifiants et contexte**

- Un identifiant a-t-il besoin d'une variable d'état extérieure pour avoir un sens ? Si oui, une référence directe de Resource élimine le problème plutôt que de le surveiller.

**État**

- L'état global est-il **un seul champ** avec **un seul point d'écriture**, ou une conjonction de drapeaux posés par plusieurs routines ?
- Un changement d'état modifie-t-il des dimensions (rayons, vitesses, plafonds) ? Si oui, sont-elles **dérivées** de l'état, ou **copiées** vers l'objet — auquel cas la liste des copies est-elle complète ?

**Initialisation et double usage**

- Une variable ou un emplacement sert-il deux rôles selon le contexte ? Si oui, sa remise à zéro est-elle **inconditionnelle**, ou dépend-elle de la donnée entrante ?
- Une valeur de remplissage ou une sentinelle peut-elle traverser une frontière de couche et y passer pour une donnée réelle ? Une sentinelle doit planter, pas s'afficher proprement.

**Champs et frontières** (le moins mécanique, et le seul que les tests attrapent)

- Deux champs du même type portent-ils des noms qui rendent une confusion évidente **à la relecture**, ou seulement après coup ?
- Un champ de donnée éditable influence-t-il réellement une sortie observable — vérifié par un test, pas par relecture ?
- Le couple `to_dict()` / `from_dict()` couvre-t-il **le même ensemble de champs** ?
