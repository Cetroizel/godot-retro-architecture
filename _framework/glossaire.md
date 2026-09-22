# Glossaire — vocabulaire technique jeu vidéo

Termes utilisés dans les décorticages sans être toujours définis sur le moment. Par ordre alphabétique — viens ici quand un terme dans un fichier de jeu n'est pas clair.

**Aliasing (d'une carte)** — quand une carte plus grande que la zone réellement chargée en mémoire « se répète » : lire une coordonnée au-delà de la fenêtre chargée renvoie ce qu'une autre portion de la carte a laissé dans la même case. Le cas F-Zero, où la tilemap Mode 7 aliase tous les 1024 pixels — invisible en version commerciale parce que la fenêtre est toujours plus large que le champ de vision.

**Banque (bank switching)** — sur les consoles 8/16 bits, le processeur ne peut adresser qu'une petite fenêtre de ROM à la fois ; un registre de la cartouche choisit quel morceau de la ROM y apparaît. Changer de banque, c'est changer la signification de toutes les adresses de cette fenêtre. Chez Metroid, chaque région du jeu est une banque : le même numéro de salle, lu dans deux banques différentes, désigne deux salles différentes — d'où le glitch des Secret Worlds.

**Bitmask (masque de bits)** — représenter un ensemble de conditions comme les bits d'un seul entier. Chez Metroid, les huit capacités de Samus tiennent dans un octet (`$01` Bombes, `$02` High Jump…) : tester une capacité est un `AND`, l'accorder un `OR`. Compact, sérialisable en un champ, et parfaitement adapté quand la relation ne porte aucun attribut propre — voir *many-to-many*.

**Block (Sonic)** — unité de collision de 16 × 16 pixels, à ne pas confondre avec la *tuile* (8 × 8, l'unité graphique du matériel). C'est le bloc, pas la tuile, qui porte un tableau de hauteurs et un angle. Chez F-Zero, le mot « block » désigne autre chose : un quartier de 256 × 256 pixels.

**Capteur (sensor)** — chez Sonic, un rayon lancé depuis le personnage vers le sol, le plafond ou les côtés pour lire la géométrie du décor. Le Sonic Physics Guide en nomme six (A à F), mais jamais tous actifs en même temps : la composition dépend de l'état (au sol, en l'air) et du quadrant du vecteur vitesse.

**Chunk (Sonic)** — bloc de décor de 256 × 256 pixels dans Sonic 1, composé de 16 × 16 *blocks*. Le niveau est une grille de chunks. (Sonic 2 et Sonic 3 & Knuckles utilisent des chunks de 128 × 128.)

**Discriminant** — l'information qui dit comment interpréter une valeur polyvalente. Explicite quand une colonne `type` accompagne une colonne `valeur` ; **implicite** — et c'est l'antipattern le plus courant du corpus — quand c'est une plage numérique (chez Zelda 1, un identifiant `< $32` désigne un type répété, `≥ $62` une liste prédéfinie) ou une variable d'état extérieure (la banque courante chez Metroid).

**DV / Stat Exp** — en Génération 1 de Pokémon, les ancêtres respectifs des IV et des EV modernes. Le DV (0–15, 4 bits par statistique) est un modificateur individuel figé ; la Stat Exp s'accumule aux combats. Les deux entrent dans la formule de calcul des statistiques ; les ignorer donne exactement le cas « DV = 0, Stat Exp = 0 ».

**ERD** (Entity-Relationship Diagram) — le type de diagramme utilisé pour montrer un modèle de données comme un schéma de base relationnelle (voir le diagramme dans l'analyse Pokémon).

**Fenêtre glissante** — ne garder en mémoire qu'une portion du décor autour du joueur, en développant juste en avance de ce qui est consommé et en recyclant derrière. Trois jeux du corpus le font, chacun sur un axe différent : Mario en colonnes, Final Fantasy en lignes, F-Zero sur les deux. C'est la brique de base de la génération procédurale d'un monde théoriquement sans limite.

**Frame** — une seule image affichée à l'écran (60 par seconde sur NES/Genesis). "À la frame exacte" veut dire un timing d'une précision de 1/60e de seconde — le niveau de précision qu'exploitent plusieurs glitches du corpus.

**FSM** (Finite State Machine, machine à états finis) — un système qui ne peut être que dans un seul état à la fois parmi un nombre fixe d'états, avec des règles explicites pour passer de l'un à l'autre. S'applique à deux échelles dans ce dépôt : la FSM globale (niveau 1 de chaque décorticage — Exploration/Combat/Menu) et une FSM locale plus petite (le pattern State pour le déroulement interne d'un combat). À noter : **aucun des huit originaux n'a de FSM globale au sens strict** — ils ont des drapeaux indépendants, et c'est la cause de plusieurs de leurs bugs.

**GoF** (Gang of Four) — les 4 auteurs du livre fondateur de 1994 sur les design patterns. "Pattern GoF" veut dire "pattern catalogué et nommé officiellement", par opposition à une solution ad hoc sans nom reconnu.

**HDMA** (Horizontal DMA) — sur SNES, un mécanisme qui écrit automatiquement dans des registres du processeur graphique à chaque ligne de balayage, sans intervention du processeur principal. C'est ce qui permet de faire varier la transformation Mode 7 ligne par ligne, et donc d'obtenir une perspective.

**Height array (tableau de hauteurs)** — chez Sonic, les 16 octets signés associés à un *block*, donnant la hauteur du sol pour chacune des 16 colonnes de pixels du bloc. Il existe un second tableau, dit « rotated », qui donne les largeurs par ligne et sert aux murs ; le plafond, lui, réutilise le tableau de hauteurs lu à l'envers.

**Hitbox** — la zone invisible utilisée pour détecter les collisions (avec un ennemi, un mur, un projectile), pas forcément identique au sprite visible à l'écran.

**Ligne de balayage (scanline)** — une ligne horizontale de l'écran, dessinée l'une après l'autre par le processeur graphique. Modifier un registre entre deux lignes permet des effets impossibles à l'image entière : c'est le principe du Mode 7, et celui des découpages raster qui séparent ciel, piste et HUD chez F-Zero.

**Many-to-many (avec ou sans payload)** — une relation où plusieurs entités d'un côté correspondent à plusieurs de l'autre. *Sans payload* : la relation ne porte rien de plus que le fait d'exister (Samus ↔ capacité) — un bitmask suffit. *Avec payload* : la relation porte un attribut propre (Pokémon ↔ capacité porte les PP) — il faut alors une **entité de jointure promue**, une classe à part entière. Le second critère de choix est la **densité** : une jointure creuse se stocke en liste, une jointure dense en matrice (les 64 coûts de sorts de Zelda II).

**Metatile** — un carré de plusieurs tuiles traité comme une unité par la logique du jeu. Chez Mario, les block buffers stockent des metatiles, pas des tuiles : c'est à cette granularité que se font les collisions. Synonyme pratique de *block* chez Sonic et de *macro-bloc* chez Metroid — chaque jeu a son nom pour la même idée.

**Metroidvania** — nom de genre, contraction de Metroid + Castlevania : une carte interconnectée explorée non linéairement, avec une progression bloquée par des capacités plutôt que par des clés ou le scénario. Ironie vérifiée : chez Metroid, **aucun de ces blocages n'est testé en code** — ils sont tous géométriques.

**Mode 7** — mode d'affichage de la SNES où une **unique** couche d'arrière-plan subit une transformation affine (rotation, mise à l'échelle, cisaillement) définie par quatre registres. En faisant varier cette transformation à chaque ligne de balayage, on obtient l'illusion d'un sol en perspective sans calculer un seul polygone. À ne pas confondre avec de la vraie 3D : Star Fox (1993) utilisait le coprocesseur Super FX pour ça, trois ans après F-Zero.

**PPU** (Picture Processing Unit) — la puce graphique dédiée du NES/SNES, ce qui affiche réellement les pixels à l'écran, distincte du processeur principal.

**RAM** (Random Access Memory) — la mémoire de travail, modifiable pendant que le jeu tourne. C'est elle qui se corrompt dans les glitches de lecture non réinitialisée (Pokémon, Metroid).

**RLE** (Run-Length Encoding) — technique de compression qui répète un octet N fois plutôt que de l'écrire N fois. Final Fantasy en utilise une variante **hybride** pour son overworld : un bit de drapeau distingue les tuiles isolées (un octet) des répétitions (deux octets), ce qui est plus économe que le RLE pur quand les tuiles isolées sont fréquentes. Zelda 1 en utilise la version minimale : **un seul bit** dans le descripteur de square, signifiant « répéter une fois ».

**ROM** (Read-Only Memory) — la mémoire du jeu qui ne change jamais : le code et les données d'origine, gravés dans la cartouche.

**Sentinelle** — une valeur réservée qui signifie « rien » ou « cas particulier » plutôt qu'une donnée réelle. Bien choisie, elle se distingue de toute valeur légitime et provoque une erreur bruyante si on l'utilise par erreur : chez Sonic, l'angle `$FF` est repérable par sa parité, seule valeur impaire du fichier. Mal choisie, elle passe pour une donnée réelle — le `$24` de remplissage de la Warp Zone de Mario, choisi parce qu'il s'affiche blanc, a fini utilisé comme numéro de monde et a produit le Minus World.

**Sequence breaking** — accéder à une zone ou un objet avant l'ordre prévu par les développeurs, généralement en exploitant un exploit de physique ou de level design plutôt qu'un vrai bug de données. Endémique dès que les verrous sont géométriques plutôt que codés (voir l'analyse Metroid).

**TAS** (Tool-Assisted Speedrun) — un parcours du jeu réalisé avec des outils (ralenti, sauvegardes d'état, frame-par-frame) pour trouver la route optimale ou étudier précisément un glitch. Source de plusieurs mécanismes de glitch documentés dans ce dépôt (tasvideos.org).

**Tile / Tileset / TileMap** — une tile est la plus petite unité réutilisable d'un décor (un carré de 8 ou 16 pixels) ; un tileset est l'ensemble des tiles disponibles pour un jeu ou une zone ; une tilemap est la grille qui indique quelle tile poser à quelle position. Attention aux unités : sur NES et Genesis la tuile matérielle fait **8 × 8 pixels**, alors que les analyses parlent souvent d'unités de 16 × 16 (voir *metatile*, *block*).

**UNION (mémoire)** — déclarer deux jeux de variables différents à la même adresse, parce qu'ils ne sont jamais utilisés en même temps. Économie de RAM classique en assembleur, et piège si rien ne vérifie quel jeu est actif : c'est le mécanisme du glitch de Mew chez Pokémon, où deux octets sont tantôt « classe et set du dresseur », tantôt « Spécial et cran d'Attaque du dernier adversaire ». L'équivalent moderne le plus proche est une table à colonnes polyvalentes.

**VRAM** (Video RAM) — la RAM dédiée à l'affichage graphique, minuscule sur les consoles rétro (quelques Ko). Sa taille limitée est à l'origine des fenêtres glissantes de Mario, Final Fantasy et F-Zero.
