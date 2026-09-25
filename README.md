# Décorticage architecture — jeux rétro vers Godot 4

Rétro-ingénierie de l'architecture technique de jeux vidéo classiques, pour ancrer des connaissances Godot 4 par la pratique plutôt que par la théorie abstraite.

> **Pour démarrer une session de travail** : [`CONTEXTE.md`](./CONTEXTE.md) rassemble la méthode, les conventions de code, les conclusions transversales et l'état d'avancement. Un seul fichier à lire plutôt que tout le corpus.

## Méthode

Chaque jeu est décortiqué selon la même grille en 4 niveaux :

1. **Machine à états globale** — les modes du jeu et les transitions entre eux
2. **Découpage des scènes** — organisation du monde et des niveaux
3. **Structures de données** — modélisation des entités, avec transposition en GDScript/Resource
4. **Design patterns observés** — et leur équivalent idiomatique Godot

Quand un bug ou glitch bien documenté illustre une leçon d'architecture, il est ajouté en illustration.

Le prompt réutilisable pour appliquer cette grille à un nouveau jeu est dans [`_framework/prompt-template.md`](./_framework/prompt-template.md). Les design patterns reconnus au fil des analyses sont centralisés dans [`_framework/design-patterns.md`](./_framework/design-patterns.md), les leçons tirées des glitches illustratifs dans [`_framework/lecons-bugs.md`](./_framework/lecons-bugs.md), le vocabulaire technique dans [`_framework/glossaire.md`](./_framework/glossaire.md), le rangement d'un projet Godot dans [`_framework/organisation-fichiers.md`](./_framework/organisation-fichiers.md), et une vue d'ensemble comparant les 8 jeux entre eux dans [`_framework/synthese-inter-jeux.md`](./_framework/synthese-inter-jeux.md).

Cette phase d'analyse se referme avec [`prochaines-etapes.md`](./prochaines-etapes.md), qui propose d'appliquer concrètement ces patterns en pratique plutôt que de continuer à empiler des analyses.

## Jeux couverts

| Jeu | Année | Plateforme | Dossier |
|---|---|---|---|
| Pokémon Rouge/Bleu | 1996 | Game Boy | [`pokemon-rouge-bleu/`](./pokemon-rouge-bleu) |
| The Legend of Zelda | 1986 | NES | [`zelda-1/`](./zelda-1) |
| Super Mario Bros. | 1985 | NES | [`super-mario-bros/`](./super-mario-bros) |
| Zelda II: The Adventure of Link | 1987 | NES | [`zelda-2/`](./zelda-2) |
| Final Fantasy | 1987 | NES | [`final-fantasy-1/`](./final-fantasy-1) |
| Metroid | 1986 | NES | [`metroid/`](./metroid) |
| Sonic the Hedgehog | 1991 | Genesis | [`sonic/`](./sonic) |
| F-Zero | 1990 | SNES | [`fzero/`](./fzero) |

## État des analyses

Les huit fichiers ont été **vérifiés contre les désassemblages communautaires** et complétés aux niveaux 3 et 4, qui étaient auparavant laissés en aperçu. Chaque fichier se termine par une section **« Corrections et ajouts »** qui liste ce qui a changé par rapport à la version initiale et pourquoi.

Les corrections les plus structurantes, si tu ne dois relire qu'une chose :

- **[Metroid](./metroid)** — Zebes n'est pas un graphe mais une **grille globale unique de 32 × 32 écrans** ; et **aucune porte ne teste une capacité en code**, tous les verrous sont géométriques.
- **[Zelda II](./zelda-2)** — « Glitch Town » n'est **pas un garde-fou volontaire** : le code ne contient aucune vérification de borne. Le bug documenté est un débordement de tableau (« healer glitch »).
- **[Super Mario Bros.](./super-mario-bros)** — le niveau est bien matérialisé en RAM, mais via **deux** buffers de 16 × 13 pour la logique de jeu et un tampon d'une colonne pour le rendu : le buffer partagé l'est entre joueur, ennemis et blocs, **pas** entre rendu et collision.
- **[Sonic](./sonic)** — il existe **deux** tableaux de collision distincts (hauteurs pour le sol, largeurs précalculées pour les murs) ; seul le plafond réutilise le tableau de hauteurs.
- **[Final Fantasy](./final-fantasy-1)** — le bug le plus illustratif n'est pas celui retenu initialement : le **taux de critique est l'index d'inventaire de l'arme**, ce qui rend l'octet de taux de critique des armes totalement mort.

Les affirmations qui n'ont pas pu être vérifiées sont désormais **signalées comme telles** dans les fichiers concernés, plutôt que laissées au même niveau de confiance que le reste — principalement chez F-Zero (statistiques par véhicule, détermination de la surface, canal HDMA) et sur deux points de Super Mario Bros. (ordre des octets d'un objet de niveau, noms des routines de collision).

## Note

Ces analyses ne reproduisent ni assets, ni code source, ni données de ROM des jeux étudiés : il s'agit d'étude architecturale originale, avec transposition pratique en GDScript pour Godot 4 — dans le même esprit qu'un article technique ou qu'un projet de désassemblage communautaire. Les extraits d'assembleur cités le sont à titre de référence courte et attribuée, comme on citerait une ligne de documentation.
