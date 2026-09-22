# Prompt réutilisable — décorticage architecture d'un jeu classique

Je suis développeur full-stack expérimenté (Java/Quarkus, Angular) en train de monter en compétence sur Godot 4. Je veux "décortiquer" l'architecture technique de [JEU] pour ancrer mes connaissances Godot par rétro-ingénierie adaptée.

Applique cette grille en 4 niveaux, sans réexpliquer les concepts OOP/patterns de base — montre directement comment ils se transposent en GDScript/Godot :

1. Machine à états globale — les "modes" du jeu et les transitions entre eux (équivalent Godot : autoload/singleton + changement de scène)
2. Découpage des scènes — un écran/une zone = une scène, ou une scène géante ? Comment les transitions sont gérées (Area2D, warps...)
3. Structures de données — comment sont modélisées les entités (personnages, objets, niveaux...), avec l'équivalent en Resource Godot (.tres) et du vrai code GDScript
4. Design patterns observés — Singleton, Factory, Strategy, Observer... et leur équivalent idiomatique Godot

Contraintes de forme :

- Progressif, pas tout d'un coup : niveaux 1 et 2 en premier, puis demande si je veux creuser 3 et 4
- Un diagramme (état, ERD...) quand ça clarifie une structure plutôt qu'un pavé de texte
- Mentionne un bug/glitch bien documenté du jeu s'il illustre une leçon d'architecture
- Relie les patterns observés à mes projets Godot en cours quand c'est pertinent
- Produire un livrable persistant (fichier Markdown) en plus de la discussion, complété au fil des niveaux
- Code GDScript : **tabulations réelles**, jamais d'espaces ; `#region`/`#endregion` sur les gros blocs

## Exigences de sourcing

Cette section a été ajoutée après une passe de vérification qui a invalidé plusieurs affirmations des premières analyses. Elle est là pour que ça ne se reproduise pas.

**Source primaire obligatoire pour toute affirmation technique.** Un mécanisme de glitch, un format de données, une adresse mémoire, une taille de structure : ça se vérifie dans un désassemblage communautaire, pas dans un wiki grand public. Les bons points d'entrée par plateforme :

| Plateforme | Sources primaires |
|---|---|
| Game Boy (Pokémon) | [pret/pokered](https://github.com/pret/pokered) et les autres dépôts pret |
| NES | [NESdev wiki](https://www.nesdev.org/wiki/), [Data Crystal](https://datacrystal.tcrf.net/), désassemblages par jeu (Zelda 1 : aldonunez ; Zelda II : FiendsOfTheElements ; Final Fantasy : BenWenger ; Metroid : nmikstas) |
| SNES | [fullsnes](https://problemkaputt.de/fullsnes.htm), [SNESdev wiki](https://snes.nesdev.org/wiki/) |
| Genesis (Sonic) | [sonicretro/s1disasm](https://github.com/sonicretro/s1disasm), [Sonic Physics Guide](https://info.sonicretro.org/Sonic_Physics_Guide) |
| Transversal | [TCRF](https://tcrf.net/), [TASVideos](https://tasvideos.org/) |

**Trois règles de rédaction qui découlent des erreurs constatées :**

1. **Citer le nom de la routine ou du label, pas seulement l'adresse.** Une adresse nue n'est pas vérifiable et se périme entre révisions du désassemblage. `wGrassRate` est utile, `$D887` seul ne l'est pas — et dans ce cas précis, l'adresse désignait le *taux* de rencontre et non la table de données, ce qui a produit une explication fausse du glitch MissingNo pendant toute la première rédaction.
2. **Vérifier l'unité avant d'écrire une dimension.** Deux erreurs de facteur 2 ont été trouvées : « écrans de 16 × 11 tiles » chez Zelda 1 (ce sont des carrés de 16 × 16 pixels, donc 32 × 22 tuiles matérielles) et « tuile » au lieu de « bloc de 16 × 16 » chez Sonic. Sur NES et Genesis, la tuile matérielle fait 8 × 8 : toute autre unité doit être nommée autrement.
3. **Distinguer « non trouvé » de « n'existe pas ».** Si une affirmation ne se vérifie pas, l'écrire explicitement plutôt que de la laisser passer ou de la supprimer en silence. Deux affirmations du corpus initial étaient inventées de bonne foi : un « écran de secours volontaire » chez Zelda II, alors que le code ne contient **aucune** vérification de borne, et une attribution nominative du qualificatif « lossy » à un développeur identifié, introuvable dans toute source consultable.

**Et une exigence de format** : terminer chaque fichier par une section **« Corrections et ajouts »** listant ce qui a changé par rapport à la version précédente et pourquoi, pour pouvoir suivre l'évolution d'une analyse sans la relire ligne à ligne.

## Questions à se poser à chaque niveau

Issues des huit décorticages — chacune a fait apparaître quelque chose qu'on n'aurait pas vu sans la poser.

**Niveau 1** — Est-ce un nouveau *mode d'interaction*, ou juste de nouvelles *données* pour le mode existant ? Et le test qui tranche les cas limites : **est-ce que la condition de sortie de l'état change ?** L'original a-t-il un état global unique, ou une conjonction de drapeaux indépendants ?

**Niveau 2** — Y a-t-il une chaîne d'indirections partagées pour le décor, et à combien de niveaux ? Une fenêtre glissante, et sur quel axe ? Les verrous de progression sont-ils vérifiés en code ou seulement géométriques ?

**Niveau 3** — Qu'est-ce qui est donnée de référence, donnée de définition, donnée d'état ? Les relations many-to-many portent-elles un attribut propre, et quel est leur taux de remplissage ? Quel est le **grain** choisi, et qu'est-ce qu'il fait perdre ? Une valeur dépend-elle d'un contexte extérieur pour avoir un sens ?

**Niveau 4** — Le comportement est-il un champ de la donnée, ou un branchement dans le moteur ? Si un état change une *dimension*, celle-ci est-elle dérivée de l'état ou copiée vers l'objet ?

## Jeux restant au backlog

Mega Man, Kirby, Castlevania, Punch-Out!!, Tetris, un metroidvania plus tardif. Voir [`../prochaines-etapes.md`](../prochaines-etapes.md) : rien n'y est urgent tant que la phase de pratique n'a pas commencé.
