# Contexte — à lire en début de session

Fichier d'amorçage de ce dépôt. Si tu es une session Claude à qui on vient de donner l'URL de ce fichier : lis-le en entier, il contient la méthode, les conventions de code et l'état d'avancement. Tu n'as pas besoin de lire les 330 Ko du corpus pour être utile — les conclusions qui comptent sont résumées ici, et les fichiers détaillés sont là si une question précise les demande.

## Le projet en une phrase

Rétro-ingénierie de l'architecture technique de huit jeux classiques, pour ancrer un apprentissage de Godot 4 dans du concret plutôt que dans de la théorie abstraite.

## Interlocuteur

Développeur full-stack expérimenté : Java/Quarkus, Angular, Hibernate/JPA en professionnel. Bases solides en OOP, design patterns et modélisation relationnelle.

Ce que ça implique pour la façon de répondre :

- **Ne pas réexpliquer les bases** OOP, patterns GoF, ni la modélisation relationnelle. Aller directement à la transposition vers GDScript.
- **Faire les ponts explicitement** vers l'équivalent Angular/Java/ORM. C'est le mécanisme d'apprentissage central de ce projet, pas une coquetterie : Scene ≈ composant Angular, Signal ≈ `@Output()`/EventEmitter, Autoload ≈ service injectable en `root`, Resource partagée ≈ entity de référence, cache du `ResourceLoader` ≈ identity map sans dirty-checking.
- **Définir le vocabulaire jeu vidéo à la première utilisation.** Pas d'acronyme non expliqué. Un glossaire est maintenu dans [`_framework/glossaire.md`](./_framework/glossaire.md).
- **Du vrai code GDScript, pas du pseudocode.** Concret et exécutable.
- **Livraison progressive** : par étapes plutôt que tout d'un bloc.

## Conventions de code — non négociables

- **Indentation : tabulations réelles, jamais d'espaces.** Godot rejette le code collé indenté avec des espaces.
- **Gros blocs et templates : organisés en régions** (`#region` / `#endregion`).
- Le vrai réflexe Godot reste cependant le **découpage en scenes/composants** dès qu'un script grossit — `#region` est utile dans un script de taille raisonnable, pas un substitut à la décomposition.
- Sauvegarde : **JSON plutôt que binaire** (lisibilité), clés centralisées en constantes plutôt qu'en dur.
- Communication entre nœuds : parent → enfant = appel direct (`$Enfant.methode()`) ; enfant → parent = **toujours par signal**.
- Composition de nœuds plutôt qu'héritage profond.

## La méthode : une grille en 4 niveaux

Chaque jeu est décortiqué selon la même grille, appliquée systématiquement :

1. **Machine à états globale** — les modes du jeu et les transitions entre eux
2. **Découpage des scènes** — organisation du monde et des niveaux
3. **Structures de données** — modélisation des entités, transposée en Resource/GDScript
4. **Design patterns observés** — et leur équivalent idiomatique Godot

Plus une section **glitch illustratif** quand un bug documenté éclaire une leçon d'architecture.

Le prompt réutilisable pour appliquer cette grille à un nouveau jeu est dans [`_framework/prompt-template.md`](./_framework/prompt-template.md), avec les **exigences de sourcing** qui en font partie intégrante : toute affirmation technique se vérifie dans un désassemblage communautaire, pas dans un wiki grand public.

## Organisation du dépôt

```
README.md                      point d'entrée et état des analyses
CONTEXTE.md                    ce fichier
prochaines-etapes.md           la sortie de la phase d'analyse vers la pratique
_framework/
├── prompt-template.md         prompt réutilisable + exigences de sourcing
├── design-patterns.md         12 patterns, avec l'idiome Godot de chacun
├── lecons-bugs.md             6 familles de bugs + check-list
├── glossaire.md               vocabulaire technique
└── synthese-inter-jeux.md     vue transversale des 8 jeux
pokemon-rouge-bleu/  zelda-1/  super-mario-bros/  zelda-2/
final-fantasy-1/     metroid/  sonic/            fzero/
```

Les huit jeux ont été **vérifiés contre les désassemblages communautaires** et complétés aux niveaux 3 et 4, qui étaient auparavant laissés en aperçu. Chaque fichier de jeu se termine par une section **« Corrections et ajouts »** qui liste ce qui a changé et pourquoi.

## Les conclusions qui comptent

De quoi répondre sans relire le corpus. Chaque point renvoie au fichier qui le développe.

### Le fil conducteur du niveau 1

**« Nouveau mode d'interaction, ou juste nouvelles données pour le mode existant ? »** — posé huit fois, avec des réponses différentes. Camp « vrai changement de mode » : Pokémon, Zelda II, Final Fantasy (tous les trois passent au tour par tour). Camp « même mode, données différentes » : Zelda 1, Mario, Metroid, Sonic (le combat reste temps réel dans la même boucle).

Le test qui tranche les cas limites, tiré de F-Zero : **est-ce que la condition de sortie de l'état change ?** Si oui, c'est une autre machine, pas un paramètre.

Nuance importante : **aucun des huit originaux n'a d'état global unique.** Ce qui en tient lieu est un ensemble de drapeaux et d'index indépendants, et la moitié des glitches du corpus vient de là. Un `enum` avec un seul champ courant et un seul point d'écriture est ce que l'original ne pouvait pas se permettre.

### La chaîne d'indirections partagées

**Cinq des huit jeux décrivent leur décor par deux à quatre niveaux d'indirection, chaque niveau étant partagé.** Zelda 1 : position → layout (121 pour 128 écrans) → colonne (150 uniques) → square. Sonic : layout → chunk 256×256 → bloc 16×16 → forme de collision. F-Zero va jusqu'à quatre niveaux. Ce n'est pas une astuce par jeu, c'est le format canonique du décor en 8 et 16 bits, et le `TileSet` de Godot en est la version moderne.

L'argument qui survit à la disparition de la contrainte mémoire : **une référence partagée est une garantie d'identité**, là où deux copies finiront par diverger.

### La fenêtre glissante

Trois jeux ne gardent en mémoire qu'une portion du décor, chacun sur un axe dicté par le mouvement : Mario en colonnes (deux buffers de 16×13), Final Fantasy en lignes (16 lignes de 256 tuiles), F-Zero sur les deux (1024×1024 px de tilemap Mode 7). Principe identique : **décompresser juste en avance de ce qui est consommé, recycler derrière**.

C'est la brique de la génération procédurale d'un monde illimité — la différence entre « niveau 1-1 en ROM » et « monde par seed » tient entièrement dans l'implémentation de la fonction qui répond à « qu'y a-t-il en (x, y) ? ». Développé dans [Mario](./super-mario-bros) et [Metroid](./metroid), niveau 4, avec les deux réserves : garantir la franchissabilité sous contrainte de capacités, et garantir le déterminisme par coordonnée.

### Verrou codé ou verrou géométrique

Le corpus contient les deux réponses opposées à la même question. [Zelda II](./zelda-2) verrouille **en code**, de quatre façons différentes (transformation de tuile, comparaison de position, entrée de table masquée, règle de franchissement). [Metroid](./metroid) ne verrouille **rien** : aucune porte ne teste un bit de capacité, tous les verrous sont géométriques — d'où le sequence breaking endémique.

Le choix n'est pas moral : un verrou géométrique récompense l'ingéniosité du joueur, un verrou codé rend le jeu testable. Ce qu'il faut éviter, c'est de croire qu'on a un verrou codé alors qu'on n'a qu'un mur.

### Les patterns les plus universels

**Flyweight**, présent dans les huit jeux sans exception, et presque toujours composé sur plusieurs niveaux. **Strategy**, présent dans les huit aussi — et ce qui le rend gratuit partout est toujours la même chose : **le comportement est un champ de la donnée de référence**, un `@export` de Resource éditable dans l'inspecteur, pas un branchement dans le moteur.

Les douze patterns relevés sont dans [`_framework/design-patterns.md`](./_framework/design-patterns.md) avec leur idiome Godot.

### Relations many-to-many : le critère de structure

Quatre jeux modélisent la même relation logique et la structurent différemment, selon deux critères seulement — **la relation porte-t-elle un attribut propre**, et **quel est son taux de remplissage** :

| Relation | Payload | Densité | Structure |
|---|---|---|---|
| Pokémon ↔ capacité | oui (PP) | creuse | entité de jointure |
| sort ↔ niveau de magie (Zelda II) | oui (coût) | dense (64/64) | matrice |
| personnage ↔ niveau de sort (FF1) | oui (charges) | grain grossier | 8 compteurs |
| Samus ↔ capacité | **non** | fini, petit | bitmask (1 octet) |

### Six familles de bugs, deux natures

Détail dans [`_framework/lecons-bugs.md`](./_framework/lecons-bugs.md). Quatre familles où **une donnée invalide est lue** — décalage de temps (Pokémon), de chemin (Mario), de plage (Zelda II), de contexte (Metroid) — et deux où **la donnée est valide et le code en tort** : mix-up de champ (Final Fantasy) et état partiellement appliqué (Sonic).

Deux points opérationnels : les bugs de Final Fantasy sont les **seuls du corpus qu'un test unitaire aurait trouvés** ; celui de Sonic est **le plus facile à écrire soi-même**, puisqu'il ne demande aucune contrainte matérielle, juste un état avec plus de conséquences qu'on n'en a listé.

### L'antipattern le plus courant des originaux

**Quatre jeux font dépendre le sens d'une valeur d'un contexte extérieur** : une plage numérique comme discriminant implicite (Zelda 1, Metroid), une variable d'état globale (la banque de ROM courante chez Metroid, la zone courante chez Sonic), ou une superposition mémoire (une `UNION` chez Pokémon). Compact, indolore à l'écriture, illisible à la relecture — et cause de la majorité des bugs.

## Corrections notables de la passe de vérification

Si une session trouve dans une source secondaire une affirmation contredite ici, c'est ici qui fait foi : ces cinq points ont été vérifiés dans les désassemblages.

- **[Metroid](./metroid)** — Zebes n'est pas un graphe mais une **grille globale unique de 32 × 32 écrans**, adressée par `(Y × 32) + X`. Le graphe est dans le level design. Et **aucune porte ne teste une capacité en code**.
- **[Zelda II](./zelda-2)** — « Glitch Town » n'est **pas un garde-fou volontaire** : le code ne contient aucune vérification de borne. Le bug documenté est un débordement de tableau que le désassemblage nomme « healer glitch ».
- **[Super Mario Bros.](./super-mario-bros)** — deux `Block_Buffer` de 16 × 13 pour la logique de jeu, un tampon d'une seule colonne pour le rendu. Le buffer partagé l'est entre joueur, ennemis, boules de feu et blocs — **pas** entre rendu et collision.
- **[Sonic](./sonic)** — **deux** tableaux de collision distincts (hauteurs pour le sol, largeurs précalculées au build pour les murs). Seul le plafond réutilise le tableau de hauteurs.
- **[Final Fantasy](./final-fantasy-1)** — le **taux de critique est l'index d'inventaire de l'arme**, ce qui rend l'octet de taux de critique des armes totalement mort et explique que Masamune ait le meilleur taux du jeu par accident.

Les affirmations qui n'ont **pas** pu être vérifiées sont signalées comme telles dans les fichiers concernés — principalement chez F-Zero (statistiques par véhicule, détermination de la surface, canal HDMA) et sur deux points de Mario (ordre des octets d'un objet de niveau, noms des routines de collision).

## Où en est la pratique

La phase d'analyse est **refermée** : la suite se joue dans Godot, pas dans de nouveaux fichiers Markdown. Voir [`prochaines-etapes.md`](./prochaines-etapes.md).

**Étape en cours** — le tutoriel officiel « Your First 2D Game » (Dodge the Creeps) est terminé, et sert de POC volontaire pour appliquer un pattern par modification :

| Modification | Pattern | État |
|---|---|---|
| Highscore persistant | — (Autoload + JSON) | fait |
| Nouveau type de mob | **Strategy** | à faire — le prochain |
| Score découplé de l'UI | **Observer** | à faire |
| Power-up temporaire | **State** | à faire |
| Difficulté progressive | **Flyweight** | à faire |
| Nouveau sprite | — (travail d'asset) | à faire |

Object Pool est volontairement exclu de cette liste : le critère n'est pas « beaucoup d'objets » mais « beaucoup d'objets **créés et détruits en rafale** » (le cas de référence étant les 32 anneaux dispersés de Sonic dans une seule frame), et Dodge the Creeps spawne ses mobs un par un.

**Étape suivante** — un RPG au tour par tour, qui réutilise directement le même outillage à plus grande échelle : le modèle `PokemonSpecies`/`Pokemon` presque tel quel, Strategy pour les cinq actions de combat, State pour le déroulement d'un tour (modèle Final Fantasy et non Pokémon, à cause du parti de 4), Observer pour l'UI, Command pour le journal de combat.

## Backlog d'analyses

Mega Man, Kirby, Castlevania, un metroidvania plus tardif. Rien d'urgent tant que les deux étapes de pratique n'ont pas transformé la théorie en code qui tourne.

## Note

Ces analyses ne reproduisent ni assets, ni code source, ni données de ROM des jeux étudiés : c'est de l'étude architecturale originale, avec transposition pratique en GDScript. Les extraits d'assembleur cités le sont à titre de référence courte et attribuée.
