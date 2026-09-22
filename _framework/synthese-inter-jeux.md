# Synthèse inter-jeux — 8 décorticages

Vue d'ensemble transversale, à lire après les fichiers individuels plutôt qu'à leur place — chaque ligne renvoie vers l'analyse complète pour le détail.

Cette synthèse a été refaite après la passe de vérification factuelle contre les désassemblages. Deux lignes du tableau ont changé de contenu (Metroid, F-Zero), et trois constats transversaux ont émergé qui n'étaient pas visibles jeu par jeu.

## Tableau comparatif

| Jeu | Année | État "Combat" séparé ? | Topologie carte/niveau | Compression | Pattern phare |
|---|---|---|---|---|---|
| [Pokémon R/B](../pokemon-rouge-bleu) | 1996 | Oui — tour par tour | Scènes séparées + warps | — | Factory, Flyweight |
| [Zelda 1](../zelda-1) | 1986 | Non — temps réel | Grille régulière (128 écrans) | Colonnes partagées + RLE 1 bit | Flyweight à 3 niveaux |
| [Mario](../super-mario-bros) | 1985 | Non — temps réel | Séquence linéaire + auto-référence (tuyaux) | Format objet (sémantique) | Fenêtre glissante |
| [Zelda II](../zelda-2) | 1987 | Oui — RPG assumé | Overworld + zones gated par objet **vérifié en code** | — | Strategy (sorts), Command |
| [Final Fantasy](../final-fantasy-1) | 1987 | Oui — parti de 4 | Overworld continu 256×256 | RLE hybride (drapeau de bit) | Many-to-many à granularité variable |
| [Metroid](../metroid) | 1986 | Non — temps réel | **Grille globale unique 32×32** — le graphe est dans le level design | Structures + macro-blocs | Flag set (bitmask) |
| [Sonic](../sonic) | 1991 | Non (sauf Special Stage) | Scroll continu, chemins multiples | Chunks → blocs → formes partagées | Flyweight géométrique |
| [F-Zero](../fzero) | **1990** | Mode course, pas de combat | Surface Mode 7 + hiérarchie à 4 niveaux | Blocks → bands → chips → tuiles | Astuce matérielle (Mode 7) |

Deux corrections par rapport à la version précédente de ce tableau : **Metroid n'a pas une topologie en graphe** au niveau du format de données (voir plus bas), et la colonne compression de Metroid, Sonic et F-Zero a été remplie — ces trois jeux compressent bel et bien, par indirection plutôt que par encodage.

## Nouveau mode ou nouvelle donnée ? Le fil conducteur du niveau 1

La question posée à chaque niveau 1 se résout en deux camps nets :

- **Vrai changement de mode** (Pokémon, Zelda II, Final Fantasy) — chaque fois qu'un jeu adopte des règles de combat au tour par tour façon RPG, l'interaction change assez radicalement pour justifier un état séparé au sens plein.
- **Même mode, données différentes** (Zelda 1, Mario, Metroid, Sonic) — dès que le combat reste temps réel et se résout dans la même boucle que le déplacement, pas besoin d'état dédié, même quand le jeu a une couche RPG en surface (Metroid a des réservoirs d'énergie, ça ne change rien à la conclusion).

F-Zero est le seul cas mixte : pas de combat du tout, mais Grand Prix et Time Trial diffèrent assez dans leurs règles pour mériter un vrai état chacun. Et son analyse fournit **le critère réutilisable** que les sept autres laissaient implicite : *est-ce que la condition de sortie de l'état change ?* En Grand Prix, on sort d'une course vers un écran de classement qui décide s'il y a une course suivante, avec un cumul à maintenir. En Time Trial, on sort vers le menu. Ce n'est pas un paramètre, c'est une autre machine.

Deux illustrations particulièrement nettes du camp « même mode » :

- **Mario** — les zones eau/souterrain/château ne sont pas des états. Et la vérification a affiné le constat : la physique n'oppose pas trois types de zone mais **l'eau à tout le reste** (souterrain et château ont la physique de l'extérieur). La gravité est une **variable RAM réécrite depuis des tables**, pas un branchement — la forme la plus pure du principe.
- **Final Fantasy** — la **même routine de décompression** sert l'overworld et les cartes de ville. Ce qui change est uniquement la stratégie d'appel : ligne par ligne au fil du déplacement pour l'un, d'un bloc à l'entrée pour l'autre.

**Nuance que la vérification a ajoutée, avec sa portée exacte** : chez Pokémon, l'état global n'est pas une valeur unique. Ce qui tient lieu de machine à états y est un ensemble de quatre drapeaux et index indépendants, posés par trois routines différentes, dont une seule les remet à zéro — et c'est la cause directe du glitch Trainer-Fly. **Ce point n'a été vérifié que pour Pokémon** : la même structure est probable chez les sept autres, vu les contraintes de l'époque, mais le corpus ne l'établit pas et il ne faut pas le lire comme tel. La leçon transposée tient indépendamment : un `enum GameState` avec **un seul** champ courant et **un seul** point d'écriture est gratuit en Godot.

## Topologies : une question d'adéquation au genre, pas de progrès

Linéaire (Mario) → grille (Zelda 1) → continu gated par objet (Zelda II) → continu large (Final Fantasy) → grille globale (Metroid) → continu multi-chemins (Sonic) → surface paramétrique (F-Zero) : ça ressemble à une escalade de complexité, mais c'est trompeur. Chaque structure sert un objectif de design précis, pas un niveau technique supérieur — la séquence linéaire de Mario n'est pas « moins avancée » que la carte de Metroid, elle est simplement la bonne réponse pour un jeu qui veut un rythme contrôlé plutôt qu'une exploration libre.

**La correction qui change cette liste** : Metroid ne relève pas d'une topologie en graphe. Zebes est **une grille globale unique de 32 × 32 écrans** (1024 octets recopiés une fois au boot en RAM de cartouche), adressée par `(Y × 32) + X`, où chaque case contient un numéro de salle et `$FF` marque une case vide. Les régions en sont des **sous-zones disjointes de la même grille**, et l'ascenseur ne change **que la banque de ROM** qui interprète les numéros — pas les coordonnées.

Le graphe est donc une propriété du **level design** — le placement des portes, des ascenseurs, des passages secrets — et pas du format de stockage. C'est plus intéressant que la version initiale, pour deux raisons :

1. **Une grille adressée par coordonnées suffit à produire une exploration non linéaire.** Il n'y a pas besoin d'une structure de nœuds et d'arêtes pour faire un metroidvania ; il faut des connexions bien placées.
2. **C'est exactement la même grille unique qui cause le glitch du jeu** (voir plus bas). La structure de données minimale qui rend le level design possible est aussi celle qui rend la corruption possible.

Deuxième constat qui n'apparaissait dans aucun fichier séparément, et qui est peut-être le plus utile du corpus. **Mario a déjà une auto-référence** : un tuyau relie une aire à une autre par pointeur, deux flux de données séparés (niveau et ennemis) étant chargés à chaque fois. C'est un embryon de graphe, deux jeux avant Metroid — sauf qu'il sert un chemin quasi linéaire. Et Final Fantasy en a une autre, appliquée non à l'espace mais aux types : la **promotion de classe** est une auto-référence `CharacterClass → CharacterClass`, réalisée par un `+= 6` sur l'identifiant.

Choisir une topologie pour un projet perso : se demander quel type d'exploration le jeu doit offrir, pas viser la structure la plus sophistiquée du corpus. Et pour un dungeon crawler grid-based — le projet visé du backlog — la leçon Metroid est directe : **une grille et une bonne table de connexions valent mieux qu'un graphe générique.**

## Verrouiller la progression : en code ou en level design ?

Le corpus contient les deux réponses opposées à la même question, dans deux jeux du même genre naissant, et le contraste est le résultat le plus exploitable de la vérification.

**[Zelda II](../zelda-2) verrouille en code**, et de quatre façons différentes : transformation de tuile par table (le Marteau change un rocher en désert), comparaison de position exacte (la Flûte), entrée de table ignorée pendant le scan (le Radeau), règle de franchissement de terrain (les Bottes). Quatre objets, quatre mécanismes, tous vérifiés par le processeur.

**[Metroid](../metroid) ne verrouille rien en code.** L'inventaire exhaustif des lectures de l'octet de capacités montre qu'elles sont **toutes** dans la physique, les dégâts ou les armes — se mettre en boule, poser une bombe, la portée du tir. **Aucune porte, aucun passage ne teste un bit de capacité.** La collision se fait par seuil d'identifiant de tuile, et le seul verrou réellement codé du jeu est le compteur de missiles des portes rouges — une ressource, pas un drapeau.

D'où le **sequence breaking** endémique au genre : un verrou posé uniquement dans la géométrie reste contournable par n'importe quel exploit de physique, et le jeu n'a aucun moyen de s'en apercevoir. L'Ice Beam en est l'illustration parfaite — un ennemi gelé devient une plateforme, donc un effet d'arme modifie la topologie franchissable, et personne n'avait croisé cet effet avec le placement de tous les blocs du jeu.

Et le plus savoureux : **c'est le jeu qui a tout mis dans le level design qui a donné son nom au genre.**

Le choix n'est pas moral. Un verrou géométrique récompense l'ingéniosité du joueur et rend le speedrun vivant ; un verrou codé garantit la progression et rend le jeu **testable**. Ce qu'il faut éviter, c'est de croire qu'on a un verrou codé alors qu'on n'a qu'un mur.

## La chaîne d'indirections partagées : le vrai format canonique

Constat transversal invisible jeu par jeu : **cinq des huit jeux décrivent leur décor par une chaîne de deux à quatre niveaux d'indirection, chaque niveau étant lui-même partagé.**

| Jeu | Profondeur | Chaîne |
|---|---|---|
| [Mario](../super-mario-bros) | 2 | objet de niveau → metatile ; plus une bibliothèque de templates de colonnes |
| [Zelda 1](../zelda-1) | 3 | position de carte → layout (**121 pour 128 positions**) → colonne (**150 uniques**) → square |
| [Metroid](../metroid) | 3 | salle → structure → macro-bloc (2 × 2 tuiles) → tuile |
| [Sonic](../sonic) | 3 | layout → chunk (**256 × 256**) → bloc (16 × 16) → forme de collision **et** 4 tuiles 8 × 8 |
| [F-Zero](../fzero) | 4 | grille 32 × 16 → block (256 px) → band (16 px) → chip (16 × 16) → 2 × 2 tuiles |

Ce n'est pas une astuce par jeu : c'est **le format canonique du décor en 8 et 16 bits**. Chaque niveau existe parce qu'il capture une échelle de répétition différente — le motif de 16 px, la bande, le quartier — et c'est pourquoi on ne peut pas les aplatir sans perdre le partage.

Le `TileSet` de Godot en est la version moderne (atlas → tuile → terrain), à ceci près qu'il ne t'oblige pas à écrire les niveaux intermédiaires.

**Deux cas valent d'être isolés.** Sonic **sépare l'identité graphique de l'identité de collision** : le même numéro de bloc indexe d'un côté une forme partagée, de l'autre quatre tuiles de rendu — donc deux blocs visuellement différents peuvent avoir la même géométrie, et un bloc peut être visible sans être solide. Et Zelda 1 ajoute une couche que les autres n'ont pas : **la recoloration**, deux bits de palette par écran suffisant à rendre le partage invisible à l'œil.

L'argument à retenir pour Godot, où la mémoire n'est plus la contrainte : **une référence partagée est une garantie d'identité.** Si deux écrans doivent rester cohérents, les faire pointer vers la même Resource rend la divergence impossible, là où deux copies finiront par diverger à la première retouche.

## Trois fenêtres glissantes, trois axes

Deuxième constat transversal. Trois jeux ne gardent en mémoire qu'une portion du décor autour du joueur, et **chacun glisse sur un axe différent** :

| Jeu | Fenêtre | Axe | Dicté par |
|---|---|---|---|
| [Mario](../super-mario-bros) | 2 pages de 16 × 13 metatiles | colonnes | le scrolling latéral |
| [Final Fantasy](../final-fantasy-1) | 16 lignes de 256 tuiles | lignes | un overworld parcouru à pied, format indexé par ligne |
| [F-Zero](../fzero) | 1024 × 1024 px de tilemap Mode 7 | les deux | une caméra qui pivote |

Le principe est identique : **décompresser juste en avance de ce qui est consommé, recycler derrière.** Et l'axe est toujours dicté par le mouvement, pas par le format.

**Précision sur Mario**, qui était le point que le brief demandait de trancher : ce ne sont pas 32 colonnes contiguës mais **deux buffers de 16 × 13** choisis selon la parité de page, et surtout — contrairement à ce qu'annonçait la version initiale — le buffer partagé ne l'est **pas entre le rendu et la collision**. Le rendu consomme un tampon d'une seule colonne ; les block buffers, dont la RAM map dit explicitement « does not effect graphics », sont partagés entre **joueur, ennemis, boules de feu et blocs**. Le fond de l'affaire reste vrai et c'est ce qui compte : **les entités n'interrogent jamais le format compressé**, le niveau est matérialisé une fois.

**Conséquence de la fenêtre chez F-Zero** : la carte **aliase tous les 1024 pixels**. Invisible en version commerciale — la fenêtre est toujours plus large que le champ de vision — mais architecturalement identique au glitch de Metroid. Un bug latent que le design rend inatteignable.

**Et c'est la brique de la génération procédurale infinie.** La différence entre « niveau 1-1 en ROM » et « monde illimité par seed » tient entièrement dans l'implémentation de la fonction qui répond à *« qu'y a-t-il en (x, y) ? »* ; l'architecture qui consomme est la même. Metroid est le terrain le plus naturel — monde adressé par coordonnées, salles résolues à la demande, cases vides déjà gérées comme un cas normal. Développé dans [Mario](../super-mario-bros) et [Metroid](../metroid) (niveau 4), avec les deux réserves qui comptent : garantir la **franchissabilité sous contrainte de capacités**, et garantir le **déterminisme par coordonnée** — sinon le monde se réécrit derrière le joueur.

## Compressions : trois stratégies, et une quatrième qu'on ne voyait pas

Même contrainte (place ROM limitée), résolue de trois façons **d'encodage** différentes :

- **[Zelda 1](../zelda-1)** partage des colonnes entre écrans, avec un **RLE à un bit** dans le descripteur de square (bit 6 = « répéter ce square une fois »). La compression la plus économe possible : un bit.
- **[Mario](../super-mario-bros)** décrit des objets sémantiques (« plateforme longueur 5 ») plutôt que des tuiles brutes, la longueur étant **encodée dans la plage du type d'objet**. C'est le plus proche de ce qu'on ferait aujourd'hui pour de la génération procédurale.
- **[Final Fantasy](../final-fantasy-1)** compresse en **RLE hybride**, et la vérification a corrigé le format : ce n'est pas « octet de tuile + compteur » systématique mais un **bit de drapeau** — octet `< $80` = tuile littérale sur **un seul** octet, `$80`–`$FE` = run avec la longueur dans l'octet suivant, `$FF` = terminateur, longueur `0` = 256. Plus économe que le RLE pur quand les tuiles isolées sont fréquentes, au prix de ne pouvoir adresser que 128 tuiles.

Et la quatrième, qui n'apparaissait pas comme une compression parce qu'elle ne ressemble pas à un encodage : **l'indirection partagée elle-même** (section précédente). Metroid, Sonic et F-Zero ne « compressent » rien au sens d'un algorithme — ils ne stockent simplement jamais deux fois le même motif. Sur les trois, c'est la stratégie la plus efficace, et c'est aussi la seule qui reste pertinente en 2026, où la place n'est plus le sujet mais la cohérence si.

À noter aussi le cas de **précalcul** de Sonic : le tableau de collision « rotated » (pour les murs) était **généré depuis des bitmaps bruts au moment du build**, par une routine restée morte dans le code. Doubler la donnée pour diviser le travail par frame — l'équivalent d'une vue matérialisée en base, et le bon arbitrage dès que la donnée est en lecture seule.

## Quatre modèles de ressource vitale

Comparaison directement utile pour concevoir la sienne :

| Jeu | Modèle | Propriété distinctive |
|---|---|---|
| [Pokémon](../pokemon-rouge-bleu) | PV | dégradation progressive, soin par objet |
| [Metroid](../metroid) | énergie + réservoirs | **plafond extensible** par collecte |
| [Sonic](../sonic) | anneaux | **tampon binaire**, tout ou rien, plafonné à 32 dispersés en pratique |
| [F-Zero](../fzero) | énergie de machine | **bouclier rechargeable par lieu** |

Deux enseignements que les chiffres exacts ont fait apparaître.

Chez Sonic, le plafond de **32 anneaux dispersés** change la lecture du système : porter 100 anneaux n'est pas plus sûr que d'en porter 32. Le compteur a deux rôles séparés — assurance-vie (bornée) et monnaie pour le Special Stage (50 requis) — et l'un des deux est silencieusement plafonné.

Chez F-Zero, le rechargement est lié à **un lieu**, pas à un objet ni à une action. Une statistique devient donc une **contrainte de parcours** : il faut *passer par* les stands, donc renoncer à la trajectoire optimale. C'est le meilleur exemple du corpus de ce qu'une ressource bien placée fait au gameplay.

## Relations many-to-many : le critère de choix de structure

Quatre jeux modélisent une relation many-to-many, et les quatre la structurent différemment — **selon deux critères seulement** :

| Jeu | Relation | Payload ? | Densité | Structure retenue |
|---|---|---|---|---|
| [Pokémon](../pokemon-rouge-bleu) | Pokémon ↔ capacité | oui (PP, PP Up) | creuse (4 sur des dizaines) | entité de jointure (`MoveSlot`) |
| [Zelda II](../zelda-2) | sort ↔ niveau de magie | oui (coût) | **dense** (64 sur 64) | **matrice** 8 × 8 |
| [Final Fantasy](../final-fantasy-1) | personnage ↔ niveau de sort | oui (charges) | dense mais **grain grossier** | 8 compteurs au lieu de 32 |
| [Metroid](../metroid) | Samus ↔ capacité | **non** | fini et petit | **bitmask** (1 octet) |

Le critère est le même qu'en base de données : **la relation porte-t-elle un attribut propre**, et **quel est son taux de remplissage** ? Pas d'attribut → un bitmask suffit. Attribut plus jointure creuse → entité de jointure. Attribut plus jointure dense → matrice. Et Final Fantasy montre le troisième levier : **choisir le grain** — huit compteurs par niveau de sort au lieu de trente-deux par sort, au prix de ne plus pouvoir épuiser un sort en particulier.

Deux pièges de modélisation repérés chez Final Fantasy, tous deux transposables tels quels :

- **La polarité des permissions.** Sa table de permission magique utilise « bit mis = **ne peut pas** lancer », ce qui rend le défaut à zéro maximalement *permissif*. Aligner le défaut du langage (zéro, vide, `false`) sur le comportement le plus restrictif est gratuit.
- **Le côté propriétaire de la relation.** La permission d'équipement n'est pas sur la classe mais **sur l'objet** (un mot de 12 bits, un bit par classe). Répondre à « que peut porter un Mage Blanc ? » exige alors de parcourir tous les objets, là où « qui peut porter cette épée ? » est une lecture directe. Le choix du côté propriétaire décide de la requête gratuite et de la requête coûteuse — et l'index inverse se construit au chargement, exactement comme le `Dictionary` d'index qu'il faut bâtir à la main pour interroger les `PokemonSpecies` par type.

## Six familles de bugs, deux natures distinctes

Détaillé dans [`lecons-bugs.md`](./lecons-bugs.md). La version précédente de cette section en comptait cinq et les ramenait toutes à une seule question. La vérification a montré que la réduction ne tient pas :

**Les quatre familles où une donnée invalide est lue** — elles se traitent par des gardes à la lecture :

| Famille | Jeu | Nature du décalage |
|---|---|---|
| Temps | [Pokémon](../pokemon-rouge-bleu) | la donnée n'a pas été réécrite depuis l'usage précédent |
| Chemin | [Mario](../super-mario-bros) | le seul chemin qui initialise a été contourné |
| Plage | [Zelda II](../zelda-2) | l'index sort du tableau |
| Contexte | [Metroid](../metroid) | l'index est **valide**, la table dans laquelle on le résout ne l'est pas |

Metroid est le seul des quatre où **la donnée lue est parfaitement valide** — lecture réussie, valeur plausible, table existante. Ce qui est faux est l'appariement, et rien dans la donnée ne permet de le détecter. C'est le cas le plus difficile, et il a la parade la plus nette : **un identifiant ne doit pas dépendre d'un contexte global pour avoir un sens** (une référence de Resource plutôt qu'un index relatif).

**Les deux familles où la donnée est valide et le code en tort** — elles se traitent par de la conception :

| Famille | Jeu | Nature |
|---|---|---|
| Code | [Final Fantasy](../final-fantasy-1) | le mauvais champ lu, une fois pour toutes, à l'écriture |
| État | [Sonic](../sonic) | un changement d'état applique certaines de ses conséquences, pas toutes |

Les trois bugs de Final Fantasy sont les **seuls du corpus qu'un test unitaire aurait trouvés** : ils sont présents à chaque exécution, sur le chemin nominal. Les quatre autres familles exigent une manipulation ou une condition particulière. Et le bug de Sonic est **le plus facile à écrire soi-même** — il ne demande aucune contrainte matérielle, juste un état avec plus de conséquences qu'on n'en a listé.

**Deux affirmations de la version précédente ont été retirées** : que « Glitch Town » serait un garde-fou volontaire de Zelda II (le code ne contient aucune vérification de borne, nulle part), et que le qualificatif « lossy » de l'état des salles de Zelda 1 serait attribuable à blargg (introuvable dans une source consultable).

**Et F-Zero n'a aucun glitch documenté**, ce qui est une conclusion et non un manque : c'est le jeu du corpus avec le moins d'état mutable, et les cinq premières familles supposent toutes un état mutable ou une table résolue dynamiquement. Corollaire directement utilisable : **la quantité d'état mutable d'un système est une bonne estimation de sa surface de bugs.**

## Le pattern le plus universel du corpus

**Flyweight**, sans exception sur les huit jeux — et presque toujours **composé sur plusieurs niveaux** (voir la section sur les chaînes d'indirection). Les species Pokémon, les 121 layouts de Zelda 1, les metatiles de Mario, les profils de terrain de Zelda II, les courbes de progression partagées entre classes de Final Fantasy, les macro-blocs de Metroid, les formes de collision de Sonic, les chips de F-Zero.

C'est le seul pattern qui traverse tout le corpus, et **le premier réflexe à adopter** — sur un projet à contrainte mémoire, mais surtout par cohérence de données.

Deuxième place, moins attendue : **Strategy**, présent dans les huit jeux aussi (effets de capacité, comportements de monstre, sorts, types d'ennemi, actions de combat, rayons, Badniks, profils de pilotage). Et ce qui le rend gratuit partout est toujours la même chose : **le comportement est un champ de la donnée de référence**, un `@export` de Resource éditable dans l'inspecteur, pas un branchement dans le moteur.

## Ce que le corpus dit de l'antipattern le plus courant

Un dernier constat, qui n'appartient à aucun jeu en particulier. **Quatre jeux du corpus font dépendre le sens d'une valeur d'un contexte extérieur**, et c'est la cause de la majorité des bugs :

- une **plage numérique** comme discriminant implicite — Zelda 1 (`< $32` = un type répété, `[$32, $62)` = objet unique, `≥ $62` = une liste), Metroid (`< $80` solide, `$80`–`$9F` destructible, `≥ $A0` traversable) ;
- une **variable d'état globale** — Metroid (la banque de ROM courante), Sonic (la zone courante pour le Collision Index), Zelda II (l'index de zone d'où sont calculés les codes de ville et de palais) ;
- une **superposition mémoire** — Pokémon (une `UNION` où deux octets sont tantôt classe/set de dresseur, tantôt Spécial/cran d'Attaque).

C'est compact, indolore à l'écriture, et illisible à la relecture. Chez Metroid, le couplage est même **double** : l'identifiant de tuile porte à la fois l'identité graphique et le comportement physique, donc ajouter une tuile destructible impose de réordonner l'atlas.

Le `TileSet` de Godot sépare nativement les deux — un `custom_data_layer` porte la propriété, l'atlas porte l'image. C'est le genre de séparation qu'on obtient gratuitement aujourd'hui et qu'il serait dommage de re-coupler par nostalgie.
