# (ARTICLE) Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks

- **url** = [article simplifié](http://algo2.iti.kit.edu/schultes/hwy/contract.pdf) (md5sum=`9dfe56278407459bb0e2d9a781bcf8f4`, [copie locale](./LOCALCOPIES/contract.pdf)) / [thèse](https://algo2.iti.kit.edu/documents/routeplanning/geisberger_dipl.pdf) (md5sum=`b31efbba33a3bc527523eed6b8722a2f`, [copie locale](./LOCALCOPIES/geisberger_dipl.pdf))
- **source** = [WEA 2008](https://dl.acm.org/doi/proceedings/10.5555/1788888)
- **auteurs** = Robert GEISBERGER (dont c'est la thèse), Peter SANDERS, Dominik SCHULTES, and Daniel DELLING / KIT
- **date de publication** = 2008
- **date de rédaction initiale de ces notes** = juin 2020 (possiblement, notes plus anciennes dans le tas)

**Notes synthétiques** :

- le travail de synthèse reste à faire
- il y a au moins une synthèse partielle pour le node-ordering, cf. plus bas

* [(ARTICLE) Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks](#article-contraction-hierarchies-faster-and-simpler-hierarchical-routing-in-road-networks)
   * [Node ordering](#node-ordering)
      * [TL;DR](#tldr)
      * [Principe général](#principe-général)
      * [Catégorie 1 = Edge difference](#catégorie-1--edge-difference)
      * [Catégorie 2 = Cost of Contraction](#catégorie-2--cost-of-contraction)
      * [Catégorie 3 = Uniformity](#catégorie-3--uniformity)
         * [Contracted neighbors](#contracted-neighbors)
         * [Sum of original edges of the new shortcuts / Hop-quotient](#sum-of-original-edges-of-the-new-shortcuts--hop-quotient)
         * [Voronoï region](#voronoï-region)
      * [Catégorie 4 = Cost of Queries](#catégorie-4--cost-of-queries)
         * [Compréhension avec les mains](#compréhension-avec-les-mains)
         * [Illustration avec la "dead-end valley"](#illustration-avec-la-dead-end-valley)
         * [Implémentation du level](#implémentation-du-level)
         * [En quoi contracter les nodes à fort level plus tard nous aide ?](#en-quoi-contracter-les-nodes-à-fort-level-plus-tard-nous-aide-)
      * [Catégorie 5 = Global measures](#catégorie-5--global-measures)
   * [Node contraction](#node-contraction)
      * [Principe général](#principe-général-1)
      * [Définitions](#définitions)
      * [Quelques notes sur la recherche des witness-paths](#quelques-notes-sur-la-recherche-des-witness-paths)
      * [Limiter le local-search](#limiter-le-local-search)
   * [VRAC À RÉORGANISER = pourquoi CH marche ?](#vrac-à-réorganiser--pourquoi-ch-marche-)
   * [VRAC À RÉORGANISER = propagation](#vrac-à-réorganiser--propagation)
      * [SOUS-VRAC À RÉORGANISER = stall-on-demand](#sous-vrac-à-réorganiser--stall-on-demand)
   * [VRAC À RÉORGANISER = Chapitre 6 = Experiments](#vrac-à-réorganiser--chapitre-6--experiments)
   * [VRAC À RÉORGANISER = Notes sur les notations utilisées](#vrac-à-réorganiser--notes-sur-les-notations-utilisées)
      * [Distance vs cost](#distance-vs-cost)
      * [Overlay graph](#overlay-graph)

## Node ordering

### TL;DR

Le principe général est de contracter séquentiellement les noeuds, le prochain noeud à ordonner est celui qui minimise (grâce à une priority-queue) un score.

Le score est une combinaison linéaire de critères. Le papier présente 8 critères, répartis selon 5 catégories :
Les 8 critères de priorité proposés sont classés en 5 catégories :

- **Edge Difference** (objectif = la CH résultante reste sparse, i.e. ne contient que peu d'edges)
- **Cost of Contraction** (objectif = diminuer le temps de preprocessing)
- **Uniformity** (objectif = répartir l'ordre de contraction des nodes sur tout le graphe)
- **Cost of Queries** (objectif = garantir que la profondeur des shortest-path-trees construits au query-time restent petite)
- **Global measures** (objectif = les noeuds "importants" du graphe (e.g. betweeness centrality) sont contractés en dernier)

Le critère présenté comme le plus important est l'edge-difference.

Note : [l'implémentation de RoutingKit](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp) utilise une combinaison linéaire de 3 critères identiques ou proches de ceux décrits plus bas :

- le level (à rapprocher du cost of queries = hierarchy-depth)
- l'edge-quotient (proche de l'edge-difference)
- le hop-quotient (à rapprocher de l'uniformity)

```cpp
return 1 + 1000*level + (1000*added_arc_count) / removed_arc_count + (1000*added_hop_count) / removed_hop_count;
```

Par ailleurs, [la thèse TCH](./2014-TDCH-thesis.md), utilise une combinaison linéaire de 4 critères, dont l'edge-quotient et la hierarchy-depth (ainsi que d'autres critères qui marchent bien avec les spécificités de TCH).

### Principe général

La partie de la thèse concernant l'ordering est le sous-chapitre **3.2 Node Order Selection** (du chapitre 3 = Cotnraction Hierarchies), entre les pages 14 et 22 incluses.

> We will first present a simple and extensible heuristic using a priority queue and then present several possible priority terms partitioned into five categories.
>
> We can chose any node order to get a correct procedure. However, this choice has a huge influence on preprocessing and query performance.
>
> The priority of a node u is the linear combination of several priority terms and estimates the attractiveness to contract this node

Le principe général est de contracter séquentiellement les noeuds, le prochain noeud à ordonner étant à chaque étape celui qui minimise (grâce à une priority-queue) un score d'ordering.

**Important** : la qualité de l'ordering a une forte importance sur la qualité de la CH résultante. La [thèse WCH](./2014-customizable-contraction-hierarchies.md) a une citation (et des références) sur le sujet :

> But finding an order which minimizes the amount of added shortcuts is known to be NP-hard [BCK+10]

*Section 3.2.1  Lazy Updates* = explications sur un détail d'implémentation pour faire l'update du score des nodes (notamment, la *dégradation* du score d'un node) le plus tard possible.

### Catégorie 1 = Edge difference

**OBJECTIF** = la CH résultante reste sparse, i.e. ne contient que peu d'edges.

*Section 3.2.2  Edge Difference* :

> Arguably the most important priority term is the edge difference. Intuitively, the number of edges in the remaining graph should decrease with the number of nodes.

Edge difference = (nombre d'edges dans le graphe APRÈS contraction de U) - (nombre d'edges dans le graphe AVANT contraction de U)

[La thèse TCH](./2014-TDCH-thesis.md), au sous-chapitre 5.2.3 Ordering the nodes, indique qu'il vaut mieux prendre l'edge-quotient (comme fait RoutingKit) plutôt que l'edge-difference :

> Note that the edge quotient works better than the more intuitive term edge difference would do. This is because the values of the difference could get so large that other terms would not have enough influence any more.

À noter qu'un critère dérivé semble être *New edges* = compter les nouveaux edges ajoutés au graphe par la contraction de U (mais d'après le papier, ce critère n'est pas intéressant, et ils l'ont ignoré).

NOTE : j'ai pas creusé, mais on dirait que (en théorie), l'edge difference de TOUS les nodes (et pas uniquement des voisins de U) peuvent être affectés par la contraction de U.
En théorie, après contraction de U, il faudrait recalculer le score d'ordering de tous les nodes du graphe. En pratique, ne recalculer que les voisins semble être une approximation acceptable. Le papier propose également de recalculer la prio de tous les nodes du graphes à certains moments clés.

### Catégorie 2 = Cost of Contraction

**OBJECTIF** = le temps nécessaire au preprocessing reste faible (ou en tout cas ne diverge pas).

*3.2.3  Cost of Contraction* :

> Those local searches to find witness paths have the biggest share of the preprocessingtime.

C'est la recherche des witness-path qui prend le plus de temps de preprocessing.

> That is the reason why we chose the sum of the search space sizes of the local Dijkstra searches for the cost of contraction for a node u, more precisely the number of settled nodes

L'idée est de dire : "si un node est lourd à contracter, on essaye de le contracter plus tardivement". Ma compréhension des choses, c'est qu'en le contractant plus tard, il y aura moins de nodes dans le graphe de contraction (vu qu'on sera plus haut dans la hiérarchie, on aura entretemps **retiré** des nodes du graphe), donc le search-space utilisé lorsqu'on cherche des witness-paths sera plus petit.

Note d'implémentation : c'est assez galère d'updater les scores des autres nodes lorsqu'on contracte un node U, puisque U peut apparaître dans le search-space size de beaucoup d'autres nodes... En pratique, donc, ils se contentent d'updater la taille du search-space-size des voisins de U.

### Catégorie 3 = Uniformity

**OBJECTIF** = répartir la contraction des nodes sur tout le graphe = éviter que des nodes proches soient contractés successivement.

*3.2.4  Uniformity* :

L'idée derrière cette catégorie de critères est que si on a un chemin linéaire (une "dead-end valley"), il faut éviter de contracter les noeuds du chemin linéaire en séquence consécutive. En effet, si contracte les noeuds consécutivement, la forward-propagation va avancer d'un noeud à la fois sur le chemin linéaire, au lieu de sauter plusieurs noeuds d'un coup grâce à un shortcut. L'idée derrière est qu'en contractant plus tardivement les nodes dont des voisins ont déjà été contractés, on tend à une profondeur de search-tree logarithmique plus tard, au query-time.

NdM = avec mes yeux naïfs, ce critère d'uniformité semble se rapprocher pas mal de la notion de hierarchy-depth, puisque dans les deux cas, on cherche à ce que les shortcuts créés "sautent" plusieurs noeuds d'un coup... Je pense que la différence est dans le moyen d'y parvenir : les critères d'uniformité sont un moyen indirect à l'ordering-time, là où la hierarchy-depth est un moyen plus directement lié à ce qui se passe au query-time (mais peut-être également plus approximé car on travaille avec l'upper-bound de la profondeur du shortest-path-tree)

#### interlude = autres notes plus anciennes sur l'importance de répartir la contraction

NdM = pourquoi est-ce important de répartir la contraction uniformément ?
- car, lorsque je suis au rank 9500 sur 10000, si les 500 nodes restants sont tous dans mon voisinage, il faudra que je les explore tous
- si à l'inverse ils sont répartis uniformément sur le graphe et que du coup je n'en ai que 20 dans mon voisinage, alors je n'aurais que 20 noeuds (au lieu de 500) à explorer pour terminer l'exploration de cette branche du graphe
- dit autrement : la répartition des noeuds du graphe permet d'élaguer rapidement des branches du graphe qu'on n'explorera pas

Encore une autre façon de voir les choses :
-  le meeting-node où se rejoignent les deux dijkstra unidirectionnels sera le node d'ordre le plus élevé sur le plus court chemin réel (qui, lui, ne dépend que du graphe original)
-  si l'ordering a regroupé TOUS les nodes d'ordre élevés à un endroit (par exemple au nord), alors on est sûr que le meeting point se trouvera au nord
-  et si on veut faire un trajet du sud au nord, le dijkstra forward qui part du sud devra explorer toute la France pour trouver son meeting-point au nord

Par ailleurs, concernant le preuve que CH accélère les queries en limitant le search space size, le papier suivant donne des bornes supérieures des search-space size pour certaines classes de graphes :
- Reinhard Bauer, Tobias Columbus, Ignaz Rutter, and Dorothea Wagner.
- Search-Space Size in Contraction Hierarchies.
- In Proceedings of the 40th Inter-national Colloquium on Automata, Languages, and Programming (ICALP’13),volume 7965 ofLecture Notes in Computer Science, pages 93–104. Springer,2013.

#### Contracted neighbors

> Contracted neighbors = count for each node the number of previous neighbors that have been contracted.

Premier critère de cette catégorie = contracted neighbours (c'est assez logique : si on juge moins prioritaire les noeuds dont les voisins ont déjà été contractés, l'ordering va naturellement se répartir sur le graphe).

#### Sum of original edges of the new shortcuts / Hop-quotient

> Count the number of original edges of the newly added shortcuts during the contraction of a node
>
> Related to the contracted neighbors counter is another priority term, it also counts the number of already contracted nodes, but now on edges. To get a better intuition for this priority term, we can also say that we count the number of edges in the original graph a shortcut represents.

Ma compréhension : chercher à minimiser le nombre de hops réels que représente un shortcut, c'est surtout un moyen de s'assurer mécaniquement de répartir la contraction sur tous les nodes du graphe (p.ex. on ne contractera un node U proche d'un node V déjà contracté *QUE* s'il ne reste plus de nodes à contracter sans voisins déjà contractés).

Ce critère m'intéresse beaucoup, car il est très proche du hop-quotient implémenté dans RoutingKit ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L563) : `(1000*added_hop_count) / removed_hop_count`).

La différence entre les deux : l'implémentation de RoutingKit ne se contente pas de minimiser le **nombre** de hops réels qu'un shortcut représente, mais plutôt le **quotient** du nombre de hops ajoutés sur le nombre de hops removed. En pratique, ça veut dire que à nombre de hops ajouté égal, on cherche à favoriser les contractions qui dégagent (par exemple par le biais de witness-path montrant que les edges ne sont pas nécessaires) les shortcuts avec beaucoup de hops réels.

Mineur : le papier parle également d'une autre motivation liée à transit-node routing (qui ne m'intéresse pas vraiment) : 

> This property should avoid shortcuts that represent too many original edges. The motivation is to use it for transit-node routing where the current implementation [35] only supportsfew node levels

Enfin, un autre avantage mentionné en passant est que si on diminue le hop-count des shortcuts, leur unpacking sera plus rapide.

#### Voronoï region

Je me suis contenté de survoler, mais en gros, la voronoi-region d'un node pas encore contracté augmente au fur et à mesure qu'on contracte ses voisins.  Du coup, il faut chercher à contracter PLUS TARDIVEMENT les nodes dont la voronoi region est la plus grande. Ça assurera qu'on répartit la contraction des nodes, et qu'on évitera de contracter des séquences linéaires de nodes.

### Catégorie 4 = Cost of Queries

**OBJECTIF** = garantir qu'au query-time, la profondeur des shortest-path-tree reste faible.

*3.2.5  Cost of Queries* :

Préambule / rappel = au query-time, la forward-propagation (resp. backward) fait grossir petit à petit un arbre des plus courts chemins à partir de la source, jusqu'à trouver le meeting-node entre forward et backward, qui a le plus haut rank du chemin final (en réalité, le critère d'arrêt d'un dijkstra bidirectionnel est un peu plus compliqué).

Ce critère vise à garantir que vu l'ordering des node choisi, la profondeur du shortest-path-tree dans le pire cas reste limitée, et donc garantir des query-time rapides.

Ma compréhension des choses, c'est que ce critère est intimement lié au critère d'uniformité. C'est d'ailleurs plutôt confirmé par cette phrase :

> The uniformity properties of the previous section are useful to speed up the queries in the resulting contraction hierarchy

Et l'ajout d'un critère explicite sur la taille des search-tree va dans le même sens :

> Adding a property that tries to limit the size of the querysearch space results in even faster query time

#### Compréhension avec les mains

D'abord avec cette quote n°1 de [la thèse sur TCH](http://digbib.ubka.uni-karlsruhe.de/volltexte/documents/3569195) :

> Only considering the removed and inserted edges, one may get quite slow queries, as the resulting hierarchy might be sparse but not flat.

Interprétation = même avec une CH qui comporte peu d'edges, on peut tout de même en nécessiter beaucoup pour calculer certains chemins ; par exemple, si on est au fond d'une impasse, et que les nodes sont en ranks croissants dans la direction de la sortie de l'impasse.

La notion de "hierarchy flat" est à prendre en analogie avec par exemple un organigramme flat dans une entreprise (citation sur [cette page wikipedia](https://en.wikipedia.org/wiki/Flat_organization) = *A flat organization (also known as horizontal organization or flat hierarchy) has an organizational structure with few or no levels of middle management between staff and executives.* ) :
- organigramme pas-flat = entre le troufion de base et le PDG, il y a beaucoup niveaux de management
- organigramme flat = entre le troufion de base et le PDG, il n'y a que quelques niveaux de management

Par analogie, une CH flat/pas-flat :
- CH pas flat = un chemin entre un node tout en bas de la hierarchy (i.e. de rank bas) et un node en haut (i.e. de rank élevé) va nécessiter beaucoup d'edges
- CH flat = un chemin entre un node tout en bas de la hierarchy (i.e. de rank bas) a tendance à être espacé d'un node en haut (i.e. de rank élevé) par peu d'edges

L'idée est donc de formaliser le fait que la hiérarchy soit flat ou non par le nombre d'edges qu'il faut pour rejoindre le haut de la hiérarchie depuis le bas de la hiérarchie.

Puis avec cette quote n°2, toujours de [la thèse sur TCH](http://digbib.ubka.uni-karlsruhe.de/volltexte/documents/3569195) :

> we preferably contract nodes that are not so far away from the bottom of the hierarchy to prevent long upward or downward paths (with “long” in terms of hops).

L'idée derrière ce critère est de nécessiter au query-time des upward-path (resp. downard) ne comportant pas beaucoup de shortcuts. Attention qu'on parle bien de l'upward-path *sur le propagation-graph* (i.e. sur le graphe contracté) et non sur le graphe original. En effet, le plus court chemin unpacked entre une source et une target, sur le graphe original est fixé (par la topologie et les poids du graphe original), l'ordering n'y changera rien.

Lien avec le critère d'uniformité : l'utilisation du `level` (décrit plus bas) conduit bien à répartir la contraction des nodes. Prenons pour exemple une CH à 10 nodes, avec un level moyen de 3 : comme on privilégie la contraction des nodes U ayant le level-max de ses voisins le plus faible, on aura tendance à aboutir à une situation où chacun des 10 nodes a un level de 3 (plutôt qu'une situation où 8 nodes sont de level 2, et 2 nodes sont de level 7).

Ce lien avec l'uniformité est rappelé par une phrase de la thèse TCH :

> Note that the way we repeatedly choose the independent set I during node ordering, helps to distribute the node contractions uniformly over the remaining graph. This should already keep the resulting hierarchy quite flat.

#### Illustration avec la "dead-end valley"

Une illustration parlante pour montrer l'intérêt de répartir uniformément la contraction des noeuds (plutôt que de les concentrer d'un côté), c'est de montrer ce qui se passe si on fait le contraire = contracter les noeuds voisins linéairement en séquence :

```
                                          |
  D                                  B    |
  x----x----x----x----x----x----x----x----|
  1    2    3    4    5    6    7    8    |
  5    1    7    3    8    4    2    6    |
```

- on a un "dead-end" (dont le dernier noeud est `D` pour dead-end) relié au reste du graphe par un bridge (noeud `B` pour bridge).
- ligne 1 = si on contracte linéairement les noeuds entre D et B sans répartir leur ordering, alors la propagation entre D et B a 8 noeuds = 12345678 (et il n'y a pas de backward-propagation).
- ligne 2 = si on répartit un peu mieux la contraction, la propagation n'a plus que 4 noeuds (forward=578, backward=68), puisqu'on est maintenant capables de "sauter" des noeuds
- dit autrement : si on contracte les nodes d'une ligne droite linéairement, l'upward path contracté ne sera pas plus petit qu'un dijkstra classique
- alors que si on contracte les nodes d'une ligne droite de façon dichotomique, l'upward-path contracté sera réduit logarithmiquement)

#### Implémentation du level

Ici, on va décrire l'implémentation du level + montrer que c'est une borne supérieure de la pire profondeur des shortest-path trees.

> We use an upper bound on the search space depth as priority term.

La notion de level (implémentée par RoutingKit, [lien1](https://github.com/phidra/RoutingKit/blob/master/src/contraction_hierarchy.cpp#L528), [lien2](https://github.com/phidra/RoutingKit/blob/master/src/contraction_hierarchy.cpp#L713), [lien3](https://github.com/phidra/RoutingKit/blob/master/src/contraction_hierarchy.cpp#L705)) est une implémentation de cette upper-bound : le level est la profondeur du pire des shortest-path-tree possibles. (dit autrement, à la propagation, lorsque `U` est settled dans le shortest-path tree, la profondeur de `U` dans ce SP-tree ne peut pas être supérieure à son `level` au moment de sa contraction, peu importe le noeud source).

**Comment c'est implémenté :**
- on maintient un attribut `level` pour chaque node du graphe, initialisé à `0`
- quand on contracte un node `U`, le level de chacun de ses voisins `V` est incrémenté à la plus GRANDE de ces deux valeurs :
    * `level(V)` : level actuel du node
    * `level(U) + 1` : level de U +1
- le level d'un node ne peut donc qu'augmenter (au fur et à mesure que ses voisins sont contractés).
- à la grosse louche, il représente le nombre de nodes contractés au max jusqu'ici dans le voisinage d'un node

**Ce qu'on va comprendre/démontrer** : si un node contracté `U` a un level `LEVEL(U)` au moment de sa contraction, alors quelle que soit la source `S` de la forward-propagation la profondeur du shortest-path-tree depuis `S` jusqu'à `U` est au maximum égale à `LEVEL(U)`.

**Point important** préalable à la compréhension = par définition, l'ordering *est* l'ordre de contraction, ce qui a des conséquences :
- au moment de sa propre contraction, un noeud `U` ne "verra" pas ses voisins qui ont été ordonnés avant lui (vu qu'ils auront été supprimés du contraction-graph)
- au moment de la propagation, un noeud `U` ne pourra donc pas se propager vers ses voisins de rank inférieurs, qui auront été contractés avant lui

**Comprendre avec les mains** pourquoi le `level` de `U` est une borne supérieure de la profondeur de tout shortest-path tree qui atteint `U` (à la propagation) ? On s'intéresse à un node `U`, au moment où on va le contracter :
- cas 1 = si ce node `U` n'a aucun voisin contracté (au moment où on contracte `U`):
    * comme le node `U` est le premier contracté de son voisinage (i.e. aucun de ses voisins `V` n'a encore été contracté), c'est que `U` aura le rank le plus petit de son voisinage
    * alors par définition, à la propagation, il n'existera **AUCUN** chemin forward *VERS* `U`, car il n'existera pas de node de rank inférieur à celui de `U`, qui pourrait avoir un edge *VERS* `U` (vu que tous les voisins de `U` seront de rank supérieurs)
    * du coup, dans tout SP-tree dans lequel `U` apparaît, `U` ne peut apparaître **QUE** en tant que racine (et son level égal à `0` est donc bien une borne supérieure de sa profondeur dans le pire SP-tree)
    * (note : ce node `U` sans voisin déjà contracté n'est pas forcément le node de rank 1, il peut très bien avoir le rank 200, si les 199 autres nodes n'étaient pas dans son voisinage...)
- cas 2 = si un node `U` a UN SEUL voisin `V` déjà contracté :
    * alors en terme de rank, on a `V < U`, et `V` est le seul voisin ayant un rank inférieur à `U`
    * du coup, dans tout SP-tree dans lequel `U` apparaît, `U` est soit racine, soit successeur de `V`
    * la profondeur de `U` dans un SP-tree quelconque est donc soit 0 (si `U` est racine du SP-tree), soit `DEPTH(V) + 1` (si `U` est successeur de `V` dans le SP-tree)
    * du coup, la borne supérieure de la profondeur de `U` dans un SP-tree quelconque est `DEPTH(V) + 1`
    * il est donc logique que lorsqu'on contracte `V`, on sette le level de `U` à `LEVEL(V) + 1`, qui est bien bien la BORNE SUPÉRIEURE de la profondeur de `U` dans tous les SP-tree possibles dans lesquels `U` peut apparaître.
- cas 3 = si un node `U` a PLUSIEURS voisins `V` déjà contractés :
    * même raisonnement que le cas 2 : dans tous les SP-tree dans lesquels `U` peut apparaître, il sera OBLIGATOIREMENT successeur d'un des `V` déjà contractés (car ce sont les seuls nodes ayant un rank `< U`, dont `U` peut-être le successeur : les autres voisins seront contractés APRÈS `U`, donc auront un rank SUPÉRIEUR à `U`)
    * du coup, la profondeur de `U` dans un SP-tree quelconque est bornée par 1 + la profondeur maximale des `V` dans le SP-tree
    * et comme à chaque fois qu'on a contracté l'un des `V`, on a mis à jour le level de `U` à `max(depth(U), 1+depth(V))`, le level de `U` a bien pour valeur cette borne supérieure, CQFD

**Preuve** du lien entre le `level` et l'upper-bound, avec le raisonnement par récurrence suivant, on montre donc que le level d'un node `U` est une borne supérieure de la profondeur à laquelle `U` peut apparaître dans un SP-tree, quel qu'il soit :
- hypothèse de récurrence = si `U` a des voisins contractés `V`, dont le level `V` est une borne supérieure de leur profondeur dans tout SP-tree, alors le level de `U` = `max(level(V) + 1)` est une borne supérieure de la profondeur de `U` dans tout SP-tree
- initialisation = si `U` n'a pas de voisin contracté, il a le level 0, et ce level est bien une borne supérieure de la profondeur à laquelle `U` peut apparaître dans un SP-tree, vu qu'il ne peut en être que la racine

#### En quoi contracter les nodes à fort level plus tard nous aide ?

> We want to reduce the depth of the shortest paths trees our modified Dijkstra algorithm grows during a query

Rappel : l'objectif est d'avoir (à la propgation) des shortest-path-tree le moins profond possible, car en moyenne pour tous les nodes du graphe, ça garantit un forward-path avec peu d'edges. (NdM = mais en pratique, ça me laisse avec une interrogation = est-ce qu'on ne répartit pas la complexité de la propagation en largeur plutôt qu'en profondeur ? ...)

> if there is a node u that is still not contracted, has a contracted neighbor v and there is a node s∈V with a large value for depth(s,v), node u should be contracted later since there is a chance that depth(s,u) = depth(s,v) + 1.

Pourquoi contracter les nodes de fort level plus tardivement aide à cet objectif ? Déjà, il faut bien voir qu'un node ayant un fort `level` va sans doute augmenter la profondeur du SP-tree à la propagation :

- à la propagation, on construit petit à petit un shortest-path tree depuis la SOURCE `S`
- supposons que juste avant de settle le node `U`, le SP-tree ait une profondeur `Dmax`
- alors il se peut que le fait de settle `U` augmente cette profondeur à `Dmax+1` : ce sera le cas si le prédécesseur `V` de `U` était une feuille du SP-tree, i.e. était sur le niveau `Dmax` du graphe
- à l'inverse, si le prédécesseur `V` n'était pas une feuille, il était à un niveau `D < Dmax` (strictement inférieur à `Dmax`), alors `U` sera au niveau `D+1`, qui sera inférieur ou égal à `Dmax` : le fait de settle `U` n'empire pas la profondeur max du SP-tree.
- donc : si `U` a un fort `level`, il aura de bonnes chances à la propagation de faire suite à un `V` déjà sur `Dmax`, et donc d'être responsable d'une augmentation de 1 de la profondeur du SP-tree.

La question est : quelle est la conséquence de contracter un `U` à fort `level` plus tardivement ?

- en étant contracté plus tardivement, `U` a moins de chances d'avoir des successeurs (en effet, si j'ai 2200 nodes en tout, le node de rank 2180 a statistiquement moins de chances d'avoir de successeurs que le node de rank 640)
- or, ces successeurs auront eux-mêmes un fort level, et seront donc probablement à leur tour responsables d'incrémenter la taille du SP-tree →  moins on en a, mieux c'est !
- dit autrement : en contractant un `U` tardivement, il a moins de chances d'avoir des successeurs dans les SP-tree, et c'est exactement ce qu'on souhaite pour les `U` à fort `level`.

C'est en tout cas mon interprétation de cette phrase :

> If node u is contracted later, it is more unlikely that there will be another node u′ that now has u as contracted neighbor with depth(s,u′) = depth(s,u) + 1.

NdM : c'est cette façon de faire qui va garantir de gros écarts de ranks entre un node et ses voisins sur le propagation-graph : en effet, on ne revient contracter un voisin du node 600 que quand on a au préalable contracté tous ceux qui avaient une depth moins importante ; du coup, quand on revient contracter les voisins du node 600, on en est maintenant au rank 1100 :-)

Dernière précision : tout ceci dépend de la profondeur des nodes dans un shortest-path-tree *D'UNE SOURCE DONNÉE*, comment on estime-ça ? Réponse = le `level` est une *borne supérieure* de la profondeur du SP-tree depuis toutes les sources possibles.

### Catégorie 5 = Global measures

**OBJECTIF** = faire en sorte que les noeuds "importants" du graphe soient contractés en dernier.

*3.2.6  Global measures* :

> We can prefer contracting globally unimportant nodes based on some path based centrality measure such as betweenness [10, 2] or reach [18].

La notion de "betweenness centrality" est très intéressante, je crois qu'elle correspond exactement à la propriété qu'on recherche dans un node classé haut :

[Page wikipedia anglaise](https://en.wikipedia.org/wiki/Betweenness_centrality) :

> In graph theory, betweenness centrality (or "betweeness centrality") is a measure of centrality in a graph based on shortest paths.
>
> For every pair of vertices in a connected graph, there exists at least one shortest path between the vertices.
>
> The betweenness centrality for each vertex is the number of these shortest paths that pass through the vertex.

[Page wikipedia française](https://fr.wikipedia.org/wiki/Centralit%C3%A9_interm%C3%A9diaire) :

> En théorie des graphes et théorie des réseaux, la centralité intermédiaire est une mesure de centralité d'un sommet d'un graphe.
>
> Elle est égale au nombre de fois que ce sommet est sur le chemin le plus court entre deux autres nœuds quelconques du graphe.
>
> Un nœud possède une grande intermédiarité s'il a une grande influence sur les transferts de données dans le réseau, sous l'hypothèse que ces transferts se font uniquement par les chemins les plus courts.


Plus généralement, la notion de "centralité" (dont il existe plusieurs variantes) indique à quel point un noeud est important dans un graphe, [lien wikipedia](https://fr.wikipedia.org/wiki/Centralit%C3%A9#Centralit%C3%A9_de_proximit%C3%A9) :

> En théorie des graphes et en théorie des réseaux, les indicateurs de centralité sont des mesures censées capturer la notion d'importance dans un graphe, en identifiant les sommets les plus significatifs.

Je ne fais que survoler cette partie, qui n'a pas l'air implémentée en pratique.

## Node contraction

### Principe général

Le principe général est de contracter les nodes séquentiellement un par un, dans l'ordre indiqué par l'ordering. L'ordering évolue au fur et à mesure de l'avancée de la contraction : à chaque étape, on contracte le node le plus intéressant de tous les nodes d'une priority-queue. Le fait de contracter un node mets à jour les poids de certains des nodes de cette priority-queue (et donc possiblement change le prochain node à contracter).

Pour contracter un node `v` :

- contracter `v` revient à le supprimer du contraction-graph, tout en ajoutant (si nécessaire) des shortcuts pour garantir que le graphe résultant a les mêmes plus courts chemins avant et après la contraction de `v`
- on itère sur toutes les paires d'edges `(e1, e2)` du produit cartésien `I x O` (où `I` représente les in-edges de `v` et `O` ses out-edges), donc les triplets de nodes `(u,v,w)`
- cas 1 = s'il existe un plus court chemin de `u` à `w` **qui ne passe pas par** `v`, appelé **witness-path**, supprimer `v` du graphe n'aura pas d'impact sur ce witness-path (car il ne passe pas par `v`), et on peut donc supprimer `v` du graphe tout en garantissant que l'ensemble des PCC du graphe sont inchangés.
- cas 2 = si le plus court chemin de `u` à `w` **passe par** `v`, il est nécessaire d'insérer un shortcut `u → w` pour que le contraction-graph "sans `v`" aie ses plus courts chemins inchangés

La section 3.3 détaille la façon dont on contracte les nodes dans l'ordre donné par l'ordering. Comme on passe beaucoup de temps à faire des local-search (recherche de ces witness-paths), la majorité de la section est consacrée à l'optimisation de cette phase, en échangeant une amélioration du temps de preprocess contre une contraction possiblement inexacte.

La section 3.4 qui suit juste derrière ne m'intéresse pas pour le moment (elle donne le lien avec Highway-Node routing, prédécesseur des CH).

### Définitions

**witness-path** = chemin prouvant (témoignant, d'où le nom de *witness*) qu'il est inutile d'ajouter le raccourci `u → w` lorsqu'on contracte `v`. Un witness-path est un plus court chemin entre `u` et `w` qui passe par un autre chemin que `u → v → w`.

**overlay-graph** = par rapport à un graphe original, un overlay-graph est un graphe avec moins de noeuds, mais préservant les plus courts chemins entre les noeuds restants.

**contraction-graph** (ma dénomination à moi) = le graphe qui évolue petit à petit au fur et à mesure de la contraction :

- avant de commencer la contraction, il est identique au graphe original
- à la fin de la contraction, il ne contient plus aucun noeud
- au moment de contracter le noeud `v`, il contient tous les noeuds de rank **supérieur** au rank de `v`, et des shortcuts supplémentaires. Tous les noeuds de rank **inférieur** ont été supprimés précédemment (lors de leur propre contraction), et on a, si nécsesaire, ajouté des shortcuts à ce moment.

**local-search** = lorsqu'on contracte un noeud `v`, c'est le fait de rechercher des witness-path pour toutes les paires `(u, w)` de prédecesseur+successeur (si on trouve un tel witness-path, il devient inutile d'ajouter un shortcut `u → w`.

### Quelques notes sur la recherche des witness-paths

Pour savoir si `v` participe à un plus court chemin ou non, inutile d'explorer tous les noeuds du graphe : il suffit de regarder s'il y a des plus courts chemins entre deux noeuds VOISINS de `v` (en effet, si un PCC de `x` à `y` passe par `v`, alors il passera forcément par deux voisins de `v`).

Le hic : comme au moment de la contraction de `v` on travaille sur l'overlay-graph (auquel certes il manque des noeuds, mais qui contient des edges supplémentaires = les shortcuts) alors les "voisins" de `v` peuvent être très TRÈS nombreux, et très éloigné les uns des autres !

> In G′, we face the following many-to-many shortest-path problem: For each source node u ∈ v+1..n with (u,v) ∈ E′and each target node w ∈ v+1..n with (v,w) ∈ E′, we want to compare the shortest-path distance d(u,w) with the shortcut length c(u,v) + c(v,w) in order to decide whether the shortcut is really needed

- on considère deux NOEUDS `u` et `w` VOISINS de `v` tel que `u → v → w` soit un chemin ; `(u,v) ∈ G′` et `(v,w) ∈ G′)` (dit autrement, on s'intéresse à tous les PRÉDÉCESSEURS `u` de `v`, et tous les SUCCESSEURS `w` de `v`)
- on veut calculer la longueur `d(u,w)` du plus court chemin `PCC(u,w)`, et la comparer avec `c(u,v) + c(v,w)` = la longueur du shortcut qu'on ajouterait en supprimant `v`
- l'idée est de NE PAS ajouter le raccourci `(u,w)` si `c(u,v) + c(v,w) > d(u,w)`, car ça voudra dire que le `PCC(u,w)` NE PASSE PAS par `v` (mais "contourne" `v` en empruntant un autre chemin)

> A simple way to implement this is to perform a forward shortest-path search in the current overlay graph G′ from each source, ignoring node v, until all targets have been found

Pour CHAQUE prédécesseur `u` de `v`, on fait un dijkstra multi-target pour calculer le plus court chemin de `u` à plusieurs noeud de `G′` (critère d'arrêt = le cost des prochains noeuds à settle est plus grand que le cost max de `max(cost(u,v)) + max(cost(v,w))`)

Note : comme on est dans l'overlay-graph, les prédécesseurs de `v` incluent les prédécesseurs via des raccourcis -> il peut y en avoir beaucoup, et très éloignés de `v` !

### Limiter le local-search

**Pourquoi limiter ?** Pour optimiser : la local-search d'un node est faite plusieurs fois : au moins une fois au moment où on contracte le node, et une ou plusieurs fois supplémentaire auparavant pour (re)calculer son score d'ordering. C'est ce qui prend le plus de temps CPU lors du preprocessing.

**Pourquoi le local-search peut prendre du temps ?** On pourrait penser que chaque noeud ayant un degré assez faible (disons inférieur à 10), explorer exhaustivement le search-space pour toutes les paires `(u,w)` est rapide ; mais en fait non !

Déjà parce qu'il suffit d'un très grand tronçon dans les in-edges ou out-edges de `v` (e.g. un tronçon ferry de 100 km) pour que le local-search de `v` aie un énorme search-space.

Et surtout : au fur et à mesure de l'avancée de la contraction on ajoute des shortcuts au contraction-graph ! Du coup, le degré moyen des nodes du graphe augmente (possiblement fortement !) au fur et à mesure de la contraction.

**Limiter ne va-t-il pas invalider la correctness de la contraction ?** Réponse courte : non.

Réponse longue = si on arrête la local-search de façon prématurée (avant qu'elle ait eu le temps de rechercher tous les plus courts chemins possibles de `u` à `w`), on va ajouter un shortcut `u → w`, et le seul risque qu'on prend, c'est de l'ajouter *à tort*, car on n'a pas pu pousser la local-search jusqu'à trouver le witness-path qui aurait prouvé que ce shortcut `u → w` était en fait inutile.

Ces shortcuts inutiles ne sont pas sans conséquences à la fois sur les perfs au query-time (car on a une CH avec un peu plus de shortcuts) et sur les perfs au preprocess-time (car avoir plus de shorcuts au moment de contracter un node N augmente le travail à faire pour contracter les nodes de rank R > N), mais le calcul des plus courts-chemins au query-time restera parfaitement correct.

**Comment limiter le local-search ?** En gros, il y a deux voies :

> Limiting the number of settled nodes is straightforward, but, in our experience, leads to more dense remaining graphs and does not speed up the contraction a lot. However, if we only use it to estimate the edge difference and the number of new edges and perform the actual contraction without limit, it speedups the node ordering without severe disadvantages.

Le reste de la section décrit des techniques avancées pour implémenter la hop-limit (1-hop limit, 2-hop limit, utilisant un bucket, etc.)

Un détail = réduction des edges on-the-fly (si on a le chemin `A--B--C--D` où `B` et `C` sont de degré 2, alors on peut remplacer ce groupe de 3 edges par un edge unique `A--D`)

## VRAC À RÉORGANISER = propagation

bidirectional dijkstra :
- forward dans G↑
- backward dans G↓
- meeting node = le noeud qui a l'ordre le plus élevé du path trouvé à la query
- (NdM : d'où l'importance de spread uniformément la contraction : si tous les noeuds d'ordre élevés sont au même endroit, ça va pas aller)

### SOUS-VRAC À RÉORGANISER = stall-on-demand

Le principe :
- À cause de la contrainte propre à CH de grimper les ranks, il se peut que la forward-propagation visite un node sans provenir d'un plus court-chemin.
- (c'est une différence importante entre la propagation dijkstra classique, et la propagation dijkstra CH)
- OR, si on visite un node en provenance d'un chemin sous-optimal, TOUS les futurs chemins relaxés depuis ce node seront sous-optimaux à leur tour !
- du coup, il est inutile de poursuivre la relaxation (et la construction d'un shortest-path tree) depuis un node qu'on a rejoint par un chemin sous-optimal : on peut "stall" le node.

Une autre façon de voir les choses :
- l'invariant classiquement maintenu par un dijkstra (bidirectionnel ou non) comme quoi un node settled a rendu définitive sa tentative_distance n'est PLUS VRAI pour une propgation CH
- en effet, avec CH, on peut settle un node "en provenance" d'un chemin sous-optimal (et le chemin optimal n'est pas emprunté à cause de la contrainte de grimper les ranks), la thèse CH a une illustration graphique assez claire de ça.
- ça n'empêche pas les CH d'être correctes :-) par contre, à moins de stall le node, ça fait explorer "pour rien" des branches de l'arbre

L'implémentation de RoutingKit aide à comprendre comment ça marche en pratique, il y a une fonction `forward_can_stall_at_node` qui renvoie un booléen [lien vers sa définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1536). Si on peut stall un node, on ne relaxe pas ses edges, [lien à l'appel](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1577) :
- avant de relaxer les edges d'un node lors de la forward-propagation on regarde ses "prédécesseurs de rank supérieur"
- (et comme on n'y a normalement pas accès à cause de la contrainte de grimper les ranks, on fait ça en regardant plutôt ses successeurs dans le graphe backward)
- si parmi ces "prédécesseurs de rank supérieur" on en trouve un déjà visité (même pas besoin qu'il soit settled !) avec une forward tentative distance inférieure à celle qu'on s'apprête à utiliser, alors c'est que le node a été visité par la forward-propagation via un chemin sous-optimal, et qu'il est inutile de le relaxer.
- (en effet, il est inutile de construire un shortest-path tree l'utilisant, puisque ce tree n'aurait justement PAS des shortest-path, à cause de la contrainte de grimper les ranks)

Mentions dans d'autres ressources :

Formulé de la façon suivante dans un autre papier sur TCH :

> During the bidirectional search we perform stall-on-demand [4, 2]: The search stops at nodes when we already found a better route coming from a higher level.

Dans [ce papier](http://algo2.iti.kit.edu/documents/routeplanning/tch_alenex09.pdf), il y a également :

> The stall-on-demand technique identifies such nodes w by checking whether(downward) edges coming into w from more important nodes give shorter paths than the (upward) Dijkstra search.

Enfin, il y a un chapitre très détaillé sur le stall-on-demand à la page 219 de [la thèse TCH](./2014-TDCH-thesis.md).


## VRAC À RÉORGANISER = Chapitre 6 = Experiments

Éléments de l'heuristique d'ordering :
- E = edge difference
- D = deleted neighbours
- S = search space size (appelé "cost of contraction" plus tôt)
- W = relative betweenness (appelé "global measures" plus tôt)
- V = sqrt(voronoi region size)
- L = limit search space on weight calculation (??)
- Q = upper bound on edges in search path (appelé "cost of queries" plus tôt)

L'average degree explose si on est trop drastique sur les hop-limits
- NdM : mon interprétation = car on rajoute des raccourcis "inutiles" -> on a plus d'edges au final
- NdM : du coup, ça semble confirmer que le nombre d'edges est bien une mesure importante pour quantifier la qualité d'un node ordering

La hop-limit est dynamique au cours de l'ordering : en fonction du degré moyen, on augmente le nombre de hops.

B/node = byte / node = space-size (taille du graph dump)

NdM = pour un nombre de nodes donné, la taille du graphe quantifie aussi la qualité de l'ordering.


## VRAC À RÉORGANISER = Notes sur les notations utilisées

### Distance vs cost

- d(u,w) = distance(u,w) = coût du plus court chemin de u à w. (u,w) n'est pas forcément un edge
- c(u,w) = cost(u,w) = poids de l'edge (u,w). (u,w) est forcément un edge.
- Même si (u,w) est un edge, on n'a PAS FORCÉMENT d(u,w) == c(u,w) :
    + sur le graphe original G, il existe peut-être un AUTRE chemin que l'edge (u,w) permettant d'aller de u à w, de façon plus courte
    + sur l'overlay graph G′, si (u,w) est un raccourci, il existe peut-être un autre chemin que le raccourci pour aller de u à w, de façon plus courte (ceci est possible si on arrêté la recherche de witness trop tôt, et qu'on a donc ajouté le raccourci à tort)

Exemple de cas où l'edge direct `AB` n'est pas le plus court chemin entre `A` et `B` : le PCC est plutôt par `A → X → B` :

```
    1.----X-----.1
    /            \
---A              B----
    \     3      /
     ------------
```
