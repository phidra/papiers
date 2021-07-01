# (ARTICLE) Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks

- **url** = [article simplifié](http://algo2.iti.kit.edu/schultes/hwy/contract.pdf) (md5sum=`9dfe56278407459bb0e2d9a781bcf8f4`, [copie locale](./LOCALCOPIES/contract.pdf)) / [thèse](https://algo2.iti.kit.edu/documents/routeplanning/geisberger_dipl.pdf) (md5sum=`b31efbba33a3bc527523eed6b8722a2f`, [copie locale](./LOCALCOPIES/geisberger_dipl.pdf))
- **source** = [WEA 2008](https://dl.acm.org/doi/proceedings/10.5555/1788888)
- **auteurs** = Robert GEISBERGER (dont c'est la thèse), Peter SANDERS, Dominik SCHULTES, and Daniel DELLING / KIT
- **date de publication** = 2008
- **date de rédaction initiale de ces notes** = juin 2020 (possiblement, notes plus anciennes dans le tas)

**Notes synthétiques** :

- CH = technique permettant de calculer des plus courts chemins dans un graphe de façon très efficace (dérivé de Dijkstra, mais beaucoup beaucoup plus efficace)
- pour cela, deux phases :
    * phase 1 = preprocess = on construit une structure de données représentant le graphe (une CH = une *Contraction Hierarchy*) qui va être utilisée pour répondre aux queries
    * phase 2 = query = on utilise la CH pour répondre de façon très efficace aux queries de plus court chemin entre deux nodes du graphe
- les notions clés sont :
    * preprocess : contraction = pour chaque noeud, ajouter des shortcuts entre ses prédecesseurs et ses successeurs, de sorte que supprimer le noeud ne modifie pas les plus courts chemins du graphe.
    * preprocess : ordering = choisir l'ordre dans lequel on contracte les nodes ; c'est là-dedans que se concentre la difficulté de la technique.
    * query-time : calcul du chemin contracté = dijkstra bidirectionnel modifié pour ajouter une contrainte = chaque sens du Dijkstra ne peux relaxer que des edges vers des nodes de rank supérieur
    * query-time : expand des shortcuts = calcul du chemin réel
- efficacité : d'autres papiers ultérieurs montrent que l'efficacité de la technique dépend du type de graphe (coup de bol : ça marche très bien sur les graphes routiers)
- technique très efficace : sur des graphe avec plusieurs dizaines de millions de nodes et edges, la query n'a besoin de settle que entre 800 et 1200 nodes dans le PIRE cas


* [(ARTICLE) Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks](#article-contraction-hierarchies-faster-and-simpler-hierarchical-routing-in-road-networks)
   * [Node ordering](#node-ordering)
      * [TL;DR](#tldr)
      * [Principe général](#principe-général)
      * [Catégorie 1 = Edge difference](#catégorie-1--edge-difference)
      * [Catégorie 2 = Cost of Contraction](#catégorie-2--cost-of-contraction)
      * [Catégorie 3 = Uniformity](#catégorie-3--uniformity)
         * [interlude = autres notes plus anciennes sur l'importance de répartir la contraction](#interlude--autres-notes-plus-anciennes-sur-limportance-de-répartir-la-contraction)
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
   * [Query](#query)
      * [Principe général](#principe-général-2)
      * [Notion à comprendre = le comportement à la query dépend de l'ordering](#notion-à-comprendre--le-comportement-à-la-query-dépend-de-lordering)
      * [Mes réflexions au sujet du critère d'arrêt](#mes-réflexions-au-sujet-du-critère-darrêt)
      * [Path reconstruction](#path-reconstruction)
      * [Stall-on-demand](#stall-on-demand)
   * [Chapitre 4 = Applications](#chapitre-4--applications)
      * [Application aux queries many-to-many = bucket-CH](#application-aux-queries-many-to-many--bucket-ch)
   * [Chapitre 5 = Experiments](#chapitre-5--experiments)
      * [setup d'évaluation (5.3 Methodology)](#setup-dévaluation-53-methodology)
      * [recherche des paramètres optimaux](#recherche-des-paramètres-optimaux)
      * [résultats](#résultats)
   * [Appendix A = Implementation et B = Code Documentation](#appendix-a--implementation-et-b--code-documentation)

## Node ordering

### TL;DR

Le principe général est de contracter séquentiellement les noeuds, le prochain noeud à ordonner est celui qui minimise (grâce à une priority-queue) un score.

Un point important à comprendre = on construit l'ordering au fur et à mesure (puisque l'ordre définitif d'un noeud dépend de la contraction des nodes qui avaient un ordre plus faible), et ce n'est qu'à la toute fin de la contraction qu'on le connaît définitivement. Dit autrement, tant qu'on n'a pas fini la contraction, on ne connait PAS (du moins, pas intégralement) l'ordering final.

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

Enfin, plus généralement, les deux idées principales derrière un ordering bien approprié sont l'edge différence et la répartition des nodes.

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

Le principe général est de contracter séquentiellement les noeuds, le prochain noeud à ordonner étant à chaque étape celui qui minimise (grâce à une priority-queue) un score d'ordering. Formulation intéressante : *the node order partition the graph into N distincts levels*.

**Important** : la qualité de l'ordering a une forte importance sur la qualité de la CH résultante. À noter qu'un mauvais ordering degradera non seulement le query-time, mais également le preprocessing-time.

La [thèse WCH](./2014-customizable-contraction-hierarchies.md) a une citation (et des références) sur le sujet :

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
En théorie, après contraction de U, il faudrait recalculer le score d'ordering de tous les nodes du graphe. En pratique, ne recalculer que les voisins semble être une approximation acceptable. Le papier propose également de recalculer la prio de tous les nodes du graphes à certains moments clés. Une autre page sur laquelle j'étais tombé (que je ne retrouve plus) suggérait également de recalculer intégralement le score d'ordering de tous les nodes (plutôt que juste l'approximer par la mise à jour des voisins des nodes contractés) lorsqu'on a contracté la moitié des nodes.

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

**overlay-graph** = par rapport à un graphe original, un overlay-graph est un graphe avec moins de noeuds, mais préservant les plus courts chemins entre les noeuds restants. On voit aussi passer le terme de **remaining-graph**.

**contraction-graph** (ma dénomination à moi) = le graphe qui évolue petit à petit au fur et à mesure de la contraction :

- avant de commencer la contraction, il est identique au graphe original
- à la fin de la contraction, il ne contient plus aucun noeud
- au moment de contracter le noeud `v`, il contient tous les noeuds de rank **supérieur** au rank de `v`, et des shortcuts supplémentaires. Tous les noeuds de rank **inférieur** ont été supprimés précédemment (lors de leur propre contraction), et on a, si nécsesaire, ajouté des shortcuts à ce moment.

**local-search** = lorsqu'on contracte un noeud `v`, c'est le fait de rechercher des witness-path pour toutes les paires `(u, w)` de prédecesseur+successeur (si on trouve un tel witness-path, il devient inutile d'ajouter un shortcut `u → w`.

**remaining graph** = l'état du graphe à l'issue de la contraction d'un node (c'est lui qui a de moins en moins de nodes).

Side-note : même si un edge `AB` existe, le plus court chemin entre `A` et `B` n'est pas forcément d'emprunter l'edge `AB`. Par exemple, dans le cas illustré ci-dessous, le PCC est plutôt par `A → X → B` :

```
    1.----X-----.1
    /            \
---A              B----
    \     3      /
     ------------
```

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

Ces shortcuts inutiles ne sont pas sans conséquences puisqu'ils dégradent à la fois les perfs au query-time (car on a une CH avec un peu plus de shortcuts) et les perfs au preprocess-time (car avoir plus de shorcuts au moment de contracter un node N augmente le travail à faire pour contracter les nodes de rank R > N), mais les plus courts-chemins calculés au query-time resteront parfaitement exacts quoi qu'il arrive. L'enjeu est donc de trouver le meilleur compromis entre la limitation du local-search et les shortcuts inutiles ajoutés.

**Comment limiter le local-search ?** En gros, il y a deux voies :

> Limiting the number of settled nodes is straightforward, but, in our experience, leads to more dense remaining graphs and does not speed up the contraction a lot. However, if we only use it to estimate the edge difference and the number of new edges and perform the actual contraction without limit, it speedups the node ordering without severe disadvantages.

Le reste de la section décrit des techniques avancées pour implémenter la hop-limit (1-hop limit, 2-hop limit, utilisant un bucket, etc.)

Un détail = réduction des edges on-the-fly (si on a le chemin `A--B--C--D` où `B` et `C` sont de degré 2, alors on peut remplacer ce groupe de 3 edges par un edge unique `A--D`)

## Query

**TL;DR** :

- dijkstra bidirectionnel sur `G↑` et `G↓`
- contrainte = grimper les ranks dans les deux sens de propagation
- il faut être vigilant sur le critère d'arrêt du bidir-dijkstra, car il est propre aux CH
- pour récupérer le chemin réel, path reconstruction = unpacker les raccourcis
- stall-on-demand pour optimiser les queries

### Principe général

À la query, le principe général est un dérivé du dijkstra bidirectionnel :

- le sens forward utilise le graphe upward `G↑`
- le sens backward utilise le graphe downward `G↓`
- **contrainte** : la propagation se fait *obligatoirement* vers les nodes de rank supérieur (c'est valable dans les DEUX sens, y compris dans le sens backward, ce qui peut sembler contre-intuitif, mais c'est en fait logique car dans un dijkstra bidir, la propagation backward sur `G↓` se fait à rebours).
- les meeting nodes (il peut y en avoir plusieurs) des deux sens de propagation seront les nodes de haut ranks
- notamment, sur le plus court chemin renvoyé au final, le meeting-node des deux sens sera le node de plus haut rank de tout le chemin (dans l'exemple ci-dessous avec `SABCDET`, il s'agit de `D`)
- le couple CH {preprocess+query} garantit de renvoyer le plus court chemin, car 1. chaque overlay-graph forward/backward conserve les plus courts chemins (grâce aux raccourcis ajoutés) et 2. si un PCC existe, les deux sens de propagation se rejoindront sur un meeting-node appartenant à ce plus court-chemin (le node de plus haut rank)
- **attention** : à cause de la façon particulière dont les CH construisent les graphes forward et backward, le critère d'arrêt du dijkstra bidirectionnel pour CH **ne peut pas être** (comme pour un dijkstra bidirectionnel classique) le fait d'avoir trouvé un meeting-node !
- à la place, on arrête un sens de propagation lorsque le cost des prochains noeuds settled sont supérieurs au cost du plus court chemin trouvé jusqu'ici (c'est un peu moins optimal que le critère classique, mais comme la propagation n'explore que log(N) noeuds, c'est pas gênant)
- last but not least, une fois le plus court chemin trouvé (il est la concaténation d'un forward-path et d'un backward-path qui 1. ont un meeting-node commun et 2. a le coût du chemin le plus petit possible parmi les candidats), on dispose d'un chemin **contracté** (i.e. certains des edges de ce chemin sont peut-être des raccourcis), il faut *unpacker* ces raccourcis (les remplacer par la liste d'edges *réels* qu'ils représentent) pour accéder au chemin réel final.

NOTE : la thèse a une façon intéressante de représenter graphiquement les ranks des nodes : plus le node est haut sur le schéma, plus son rank est elevé (ça évite de surcharger les schémas en annotant les ranks).

### Notion à comprendre = le comportement à la query dépend de l'ordering

- étant donné un OriginalGraph, le plus court chemin entre `S` et `T` est fixé (et ne dépend que du graphe), par exemple `S-A-B-C-D-E-T`
- ce plus court chemin existe et est fixé, indépendamment de l'ordering qu'on va choisir, ou bien de la CH qu'on s'apprête à construire
- notamment, lorsqu'on fait l'ordering des nodes du graphe, chacun des nodes `SABCDET` se voit attribuer un rank, par exemple :
```
S   A   B   C   D   E   T
8   21  10  3   48  19  14
```
- si on s'intéresse à ce PCC précis, ce qui se passera au query-time est fixé par cet ordering :
    + notamment, le meeting-node des deux dijkstra sur ce chemin sera le node du chemin ayant le rank le plus élevé (ici, `D=48`).
    + de même, le forward-path sur ce PCC précis est fixé par l'ordering et sera `S-A-D` (les nodes `B` et `C` n'y apparaissent pas car ils font partie du shortcut `A-D`)
    + de même, le backward-path sur ce PCC précis est fixé par l'ordering et sera `T-E-D`
- **conclusion** = même si le plus court chemin entre `S` et `T` sera toujours le même quoi qu'on fasse (`SABCDET`), les caractéristiques des queries dépendent de l'ordering.

(NdM : d'où l'importance de répartir uniformément la contraction sur tout le graphe : si tous les noeuds d'ordre élevés sont au même endroit, ça va pas aller)

### Mes réflexions au sujet du critère d'arrêt

**TL;DR** : en gros, je ne suis pas encore tout à fait convaincu qu'on ne puisse pas arrêter le dijkstra dès qu'on a trouvé un meeting-node *en théorie*, mais ça paraît compliqué *en pratique*.

> abort the forward/backward search process when all keys in the respective priority queue are greater than the tentative shortest-path length (abort-on-success criterion). Note that we are not allowed to abort the entire query as soon as both search scopes meet for the first time.

Section 3.5, figure 20, page 29 = exemple de cas illustrant qu'il ne faut pas arrêter le dijkstra dès qu'on a trouvé un meeting-node (comme on l'aurait fait sur un dijkstra bidir classique).

NdM : je suis pas encore tout à fait convaincu, on dirait qu'en itérant sur tous les successeurs des nodes déjà settled (comme dans un dijkstra bidir classique), ça pourrait passer ?

Hum, peut-être que c'est compliqué d'itérer sur les successeurs des nodes déjà settled (car il faut itérer sur leur successeurs ET prédecesseurs dans G↑ et G↓)

Dans l'exemple donné en illustration en tout cas, à supposer que les arcs soient directionnels (s -> y -> t  ;  s -> x -> t) :

Du point de vue de S :
- y n'est pas un successeur de s dans le graphe forward (car y a un rank inférieur à s)
- y n'est pas un prédecesseur de s dans le graphe forward (car l'arc est orienté s->y dans le graphe forward)
- y **EST** un prédecesseur de s dans le graphe backward (car y a un rank inférieur à s ET l'arc va dans le bon sens dans le graphe backward)
- y n'est pas un successeur de s dans le graphe backward (car l'arc est orienté y->s dans le graphe backward)

Du point de vue de T :
- y n'est pas un successeur de t dans le graphe forward (car l'arc est orienté y->t dans le graphe forward)
- y n'est pas un prédécesseur de t dans le graphe forward (car y a un rank supérieur à t)
- y **EST** un successeur de t dans le graphe backward (car y a un rank supérieur à t ET l'arc va dans le bon sens dans le graphe backward)
- y n'est pas un prédécesseur de t dans le graphe backward (car l'arc va dans le mauvais sens dans le graphe backward)

Du coup, j'ai quand même l'impression qu'en itérant sur tous les settled nodes, on pourrait trouver la liste des meeting-nodes en regardant leurs successeurs/prédécesseurs... Mais peut-être est-ce trop compliqué en pratique car il faudrait accéder aux prédécesseurs d'un node, ce qui n'est pas facile ?

De plus, vu que les noeuds settled par une query CH sont en log du nomber de noeuds, on n'a encore moins envie d'aller vers cette complexité.

### Path reconstruction

**TL;DR** : le chemin est calculé sur la CH, il est constitué de shortcuts → pour reconstituer le chemin final, il faut unpacker les shortcuts, i.e. remplacer chaque shortcut par la concaténation de ses deux demi-edges, jusqu'à ce qu'on n'ait plus que des demi-edges originaux (i.e. non-shortcuts) dans le chemin.

Section 3.5.1, figure 22, page 31, illustration assez clair des différentes étapes pour unpacker un chemin contracté.

Unpacking récursif de chaque shortcut (un shortcut est toujours l'agrégat de deux demi-edges, qui peuvent éventuellement être des shortcuts eux aussi).

Mentionne l'AdjacencyArray pour stocker `G↑` et `G↓` (cf. aussi [la doc de RoutingKit](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/SupportFunctions.md) qui présente la structure de façon un peu plus détaillée).

Pour pouvoir unpacker un shortcut, il est indispensable de stocker de l'info en plus (car vu que le middle-node est le node de rank le plus petit, aucun des deux demi-edges n'existe dans l'AdjacencyArray stockant `G↑` et `G↓` ; dit autrement, la CH n'a **pas** de base toute l'info nécessaire pour unpacker les shortcuts). Le plus simple est de stocker comme info en plus : le middle-node de chaque shortcut.


### Stall-on-demand

**TL;DR** : c'est une technique d'optimisation de la query (spécifique à CH car le *besoin* d'optimisation provient des contraintes de grimper les ranks) permettant de couper des branches de l'arbre d'exporation dijkstra, car on peut savoir à l'avance qu'ils sont sous-optimaux. Les gains ne sont pas négligeables : la section 5.4.7 indique qu'elle permet de diviser par 3 le query-time !

Note = ce qui suit s'applique aux deux sens forward et backward, mais pour ne pas alourdir les notes, je ne parle que de la forward-propagation.

Le principe :
- À cause de la contrainte propre à CH de grimper les ranks, il se peut que la forward-propagation visite un node sans provenir d'un plus court-chemin (la thèse a une illustration du problème, qui est très claire une fois qu'on a compris le principe, et je détaille ma propre illustration ci-dessous)
- (c'est une différence importante entre la propagation dijkstra classique, et la propagation dijkstra CH)
- OR, si on visite un node en provenance d'un chemin sous-optimal, TOUS les futurs chemins relaxés depuis ce node seront sous-optimaux à leur tour !
- du coup, il est inutile de poursuivre la relaxation (et la construction d'un shortest-path tree) depuis un node qu'on a rejoint par un chemin sous-optimal
- le stall-on-demand consiste à interrompre la relaxation depuis un node qu'on sait sous-optimal (on peut "stall" le node)

Une autre façon de voir les choses :
- l'invariant habituellement maintenu par un dijkstra "normal" (bidirectionnel ou non) comme quoi la tentative-distance d'un node devient son coût optimal lorsqu'on le *settle* ... **N'EST PLUS VRAIE** pour une forward-propgation CH
- même rendue définitive car on a settle le node, la tentative-distance d'un node peut être sous-optimale
- en effet, avec CH, on peut settle un node "en provenance" d'un chemin sous-optimal (le vrai chemin optimal pour rejoindre le node n'est pas emprunté à cause de la contrainte de grimper les ranks), cf. illustration dans la thèse ou ci-dessous
- ça n'empêche pas les CH d'être correctes et de renvoyer le bon itinéraire :-) par contre, à moins de stall le node, ça fait explorer inutilement des branches de l'arbre
- le stall-on-demand consiste à détecter, et ne pas explorer inutilement ces branches

**Illustration graphique du problème :**

Pour reprendre l'illustration de la figure 24, le graphe upward `G↑` ci-dessous comporte 3 noeuds `S`, `U` et `X` dont l'ordering est `S < U < X` (ici, `1 < 2 < 3`) :

```
(1)          5         (3)
 S─────────────────────►X
 │                      ▲
 │          (2)         │
 └─────────► U ◄────────┘
     100     │     5 
             │
             │         (4)
           42└────────► K
```

On suppose qu'on fait une forward-propagation depuis `S` :

- vu l'ordering, depuis `S`, on peut rejoindre `U` ou `X` qui sont tous deux de ranks supérieurs
- la propagation va commencer par settle `X`, dont la distance prend sa valeur définitive de `5`
- plus tard, on va settle `U`, dont la distance sera définitive à `100`... alors qu'on peut clairement rejoindre `U` par un chemin plus court `S -> X -> U` de poids `10` au lieu de `100` !
- dit autrement, en rejoignant `U` depuis `S`, on a emprunté un chemin non-optimal...
- ...le chemin optimal `S -> X -> U` ne sera jamais emprunté par cette forward-propagation, car au vu de l'ordering `U < X`, l'edge intéressant `XU` (intéressant car de poids faible) ne sera jamais relaxé dans le forward-search
- et sans le *stall-on-demand*, on aurait continué la forward-propagation depuis `U` sur une base biaisée (en supposant que la distance de `U` est `100`)
- par exemple, on aurrait settle `K` avec un coût total depuis `S` valant `142`, alors que le vrai coût de `K` depuis `S` n'est que de `52` !

Ce défaut ne change pas la correctness finale de l'algo (qui trouvera le chemin optimal quoi qu'il arrive), mais explorer les successeurs de `U` en considérant que la distance de `U` est `100` ne sert à rien et fait consommer du temps CPU complètement inutile.

**Principe** = le *stall-on-demand* consiste à détecter ces situations (où la forward-propagation a rejoint un node par un chemin non-optimal, mais qu'un chemin plus efficace ne sera jamais relaxé à cause du rank de ses noeuds), et à *stall* (i.e. arrêter) la forward-propagation depuis `U`.

L'implémentation de RoutingKit aide à comprendre comment ça marche en pratique, il y a une fonction `forward_can_stall_at_node` qui renvoie un booléen [lien vers sa définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1536). Si on peut stall un node, on ne relaxe pas ses edges, [lien à l'appel](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1577) :
- lors de la forward-propagation, avant de relaxer les edges d'un node `N`, pour chaque noeud de destination, on regarde ses "prédécesseurs de rank supérieur"
- (comme on n'y a normalement pas accès sur le graphe forward `G↑` à cause de la contrainte de grimper les ranks, pour réussir ça, on regarde plutôt les successeurs de `N` dans le graphe backward `G↓`)
- si parmi ces "prédécesseurs de rank supérieur" on en trouve un déjà visité par la **forward**-propagation (même pas besoin qu'il soit settled), qui a une forward tentative distance telle qu'on peut rejoindre `N` avec un coût inférieur à celui qu'on s'apprête à utiliser, alors c'est que `N` a été visité par la forward-propagation via un chemin sous-optimal, et qu'il est inutile de poursuivre la forward-propagation depuis `N`.
- (en effet, il est inutile de construire un shortest-path tree depuis `N`, puisque ce tree n'aurait justement PAS des shortest-path, à cause de la contrainte de grimper les ranks)

Note : attention à ne pas se mélanger les pinceaux : dans notre exemple, on ne s'intéresse qu'à la forward-propagation (le graphe backward n'est qu'un détail d'implémentation permettant de connaître facilement les prédécesseurs d'un node). Notamment, c'est bien si un prédécesseur `P` a déjà été visité par la **forward**-propagation qu'on regarde si `PN` n'est pas un chemin plus optimal vers `N`.


**Application à mon illustration ci-dessus** :
- comme en temps normal, la forward-propagation a relaxé `U` avec un coût depuis `S` à `100`
- puis, toujours comme en temps normal, on settle également `X` avec un coût depuis `S` à `5`
- plus tard, arrive le moment de settle `U`, qui a un coût de `100`
- ce qui change : avant de relaxer les out-edges de `U`, on regarde **dans le graphe backward** les successeurs de `U` : on trouve `X`.
- `X` a déjà été visité par la **forward**-propagation, et a un coût définitif depuis `S` de `5`. De plus, l'edge `XU` a un coût de `5` également, ce qui fait que depuis `S`, on peut rejoindre `U` (via `X`) avec un coût total de `10`.
- comme `10 < 100`, c'est donc que le chemin direct `SU` par lequel la forward-propagation a pu rejoindre `U` n'était pas optimal -> il est inutile de continuer la propagation depuis `U`.
- on peut donc settle un autre node (non-illustré sur le schéma), et la forward-propagation ne settlera jamais `K` ! C'est via la backward-propagation que `K` sera settled à sa vraie valeur.

À noter qu'il y a un chapitre très détaillé sur le *stall-on-demand* à la page 219 de [la thèse TCH](./2014-TDCH-thesis.md).

*Note* = à ne pas confondre avec *stall-in-advance* = une autre technique provenant de Highway Node Routing pour limiter les local-searches (qui semble équivalente à la hop-limit).

*Note* : la représentation graphique où les nodes apparaissent plus ou moins haut dans la figure en fonction de leur rank est intéressante : elle permet d'éviter d'alourdir l'illustration en annotant le rank de chaque node, je la garde sous le coude pour communiquer sur l'ordering par la suite.


## Chapitre 4 = Applications

Une section assez courte sur les applications :

- **4.1 Many-to-Many Routing** : application la plus intéressante à mes yeux (et utilisée par l'algorithme ULTRA), je détaille un chouïa plus bas.
- **4.2 Transit-Node Routing** : utiliser l'ordering pour choisir les access-nodes de TNR
- **4.3 Changing all Edge Weights** : moins intéressant que CCH
- **4.4 Implementation on Mobile Devices** : application intéressante des CH, qui sont efficaces et peu gourmandes en RAM
- **4.5 Reuse of Shortcuts in other Applications** : quelques mots seulement.

### Application aux queries many-to-many = bucket-CH

**Objectif** = calculer tous les shortest-paths entre toutes les sources `s` d'un ensemble `S` et toutes les targets `t` d'un ensemble `T`.

- implémentation naïve = faire `|S| x |T|` queries. Simple, mais impraticable pour de grandes valeurs de `|S|` et `|T|`
- utilisation des CH = à la place, on peut faire `|T|` backward-searches + `|S|` forward-searches, soit `|S| + |T|` searches en tout (chaque search étant de plus assez rapide, car faite sur une CH).

**Step 1** = on fait `|T|` backward-propagations :

- pour chaque `t ∈ T`, on fait une backward-propagation complète (i.e. sans critère d'arrêt : ce n'est que lorsqu'on a épuisé les noeuds à settle qu'on s'arrête)
- on associe à chaque node `n` du graphe un bucket `Bn` contenant `|T|` valeurs
- lorsqu'une backward-propagation reach un node `n`, on sette dans `Bn[t]` le cost avec lequel la propagation a atteint `n` (et on aura `Bn[t] = +∞` si `n` n'a pas été reached par la backward-propagation depuis `t`)
- au final, *chacune* des backward-propagations sette une valeur aux noeuds qu'elle atteint (inutile de les settle : simplement les atteindre est suffisant)

**Step 2** = on fait `|S|` forward-propagations :

- derrière, pour chaque `s ∈ S`, on fait une forward-propagation
- lorsque la forward-propagation partant de `s` atteint un node `n` déjà atteint par la backward-propagation depuis `t` (i.e. on a `Bn[t] ≠ ∞`), tout se passe comme si `n` était un meeting-node pour ce couple `{s,t}` : on a alors un plus-court-chemin candidat.
- comme on a fait une backward-propagation pour chaque `t ∈ T`, et une forward-propgation pour chaque `s ∈ S`, pour chaque couple `{s, t}`, on peut trouver le plus court chemin optimal parmi les candidats.

Note = on arrête les propagations quand ?

- Avec ma compréhension, on n'arrête *jamais* les backward-propagation, qui doivent reach tous les noeuds possibles (mais comme c'est une propagation CH, elles ne devraient pas être très coûteuses)
- Du côté des forward-propagations, en théorie, on pourrait s'arrêter si le critère d'arrêt du bidir dijkstra des CH est rempli pour tous les noeuds `s ∈ S` (i.e. pour tous les `s ∈ S`, les noeuds non-encore settled ont un coût plus important qu'un plus court chemin candidat)
- En pratique, vu le caractère sparse des propagations CH, c'est plus simple, et sans doute pas énormément plus lent de propager complètement les forward-propagation...

Note bis = pourquoi est-il nécessaire de faire *plusieurs* forward-searches ?

- Avec une compréhension naïve, on pourrait penser qu'il suffit de se contenter de regarder les buckets des voisins immédiats de `s`(voire de regarder juste le bucket de `s` lui-même `Bs`).
- En effet, si tous les voisins `v` de `s` ont une valeur `Bv[t]` pour tous les `t`, alors il suffit de sommer les cost `cost(v) + Bv[t]` et d'en prendre le min pour trouver le plus court chemin ?
- Non ! Car pour une backward-propagation donnée, tous les noeuds du graphe ne sont pas reached, loin de là (c'est justement l'intérêt des CH de ne pas devoir relaxer tous les noeuds)
- Du coup, parmi les voisins `v` de `s`, il y en a sans doute qui ont `Bv[t] = +∞` : il faut continuer la propagation plus profondément pour trouver un node `n` atteint par la backward-propagation depuis `t` (dont `Bn[t] ≠ +∞`)
- Plus généralement, il faudra avancer chaque forward-propagation pour réussir à tomber sur des noeuds atteints par toutes les backward-propagations (jusqu'à trouver des meeting-nodes, en fait, et ce pour *chaque* `t∈T`)
- Par exemple, il est tout à fait possible qu'aucun voisin de `S` n'ait été atteint par aucune des backward-propagation (dit autrement : on peut très bien avoir `Bv[t] = ∞` pour chaque voisin `v`)

Note ter : du coup, dans les backward-propagation, on peut ignorer les stalled-nodes, et faire comme si ils n'avaient pas été reached (vu qu'ils ne participeront pas aux plus courts chemins).

**Optimisation** : la thèse mentionne une optimisation consistant à prune le backward-search pour avoir un bucket plus petit, elle semble décrite plus en détail sur le papier de 2007 dont est issue l'algo many-to-many = *Computing many-to-many shortest paths using highway hierarchies* ([lien](https://dl.acm.org/doi/10.5555/2791188.2791192), [PDF](https://dl.acm.org/doi/pdf/10.5555/2791188.2791192)).


## Chapitre 5 = Experiments

Il parle un peu de l'implémentation (mais comme on n'a pas accès au code-source, ça n'a que peu d'intérêt ; en particulier, l'appendix B est quasi-inutile vu qu'elle donne des schémas UML sur un code que je n'ai pas). Info utile toutefois : son implémentation des CH fait 7700 lignes de code, et il compare les résultats à un dijkstra classique pour s'assurer de la correction des algos.

Il utilise 3 datasets :

- Europe = 18M nodes, 42M edges
- USA = 23M nodes, 58M edges (librement accessible)
- New-Europe = 34M nodes, 75M edges

À noter qu'ils n'avaient pas accès au travel-time des edges, et qu'ils l'ont donc estimé en fonction de la catégorie de la route. De plus, ils se sont limités à la plus grande composante fortement connexe.

L'occupation mémoire (ou plus exactement l'overhead-mémoire par rapport à un dijkstra unidirectionnel) est mesuré en bytes-per-node.

### setup d'évaluation (5.3 Methodology)

Il y a 3 setups permettant d'évaluer comment les CH se comportent :

**setup de base** : le setup de base pour évaluer la performance des CH est :

- 100.000 OD (Origin-Destination) choisies uniformément au hasard, et 3 critères :
- Average query-time
- Average search-space size
- Average relaxed edges

**setup par dijkstra-rank** : setup différent, et plus intéressant car tenant compte de la difficulté d'une query :

- on prend 1000 sources `s` random, et on fait une propagation Dijkstra (classique, pas CH) partant de `s`
- on retient le noeud settled au rang `r` pour toutes les différentes puissances de 2 : `r = 2^k`
- chacun de ces noeuds settled `t` fournit une requête `{s,t}`, de plus en plus difficile
- exemple 1 = pour une source `s`, le noeud `t₃` settled au rang `2³ = 8` fournit une requête `{s,t₃}` très facile (vu que le chemin est tout petit, il sera hyper-rapide à calculer même pour un dijkstra classique, car il ne nécessite de settle que `8` noeuds)
- exemple 2 = pour la même source `s`, le noeud `t₁₂` settled au rang `2²¹ = 2097152` fournit une requête `{s,t₂₁}` très difficile (vu que le chemin est très long : avec un dijkstra classique, il nécessite de settle plus de 2 millions de nodes)
- cette façon de faire permet d'évaluer la performance des algos en fonction de la difficulté des requêtes

**setup par worst-case-upper-bound** : le principe est de raisonner sur le pire cas possible, en calculant une upper-bound de la taille du search-space (taille totale = taille forward + taille backward) pour toute s-t-pair. Pour cela :

- sur un graphe `G`, on fait une forward (resp. backward) search complète (i.e. sans abort criterion : on ne s'arrête que quand la queue est vide) depuis chacun des nodes
- on obtient une distribution des tailles des forward-search-space (resp. backward) sur le graphe `G`
- on peut donc attribuer à chaque node une upper-bound sur un forward-search-space depuis ce node (et de même, on peut attribuer à chaque node une upper-bound sur un backward-search-space depuis ce node)
- ce qu'il faut retenir : on peut calculer pour s-t-pair une upper-bound du search-space total (i.e. de la somme du forward-search-space et du backward-search-space), en n'ayant fait que `2*N` propagations, au lieu d'avoir dû en faire `N²`

Ça permet de mesurer la performance des CH dans le pire cas (et même de deux variantes des CH, en fonction des critères d'ordering) et de comparer ça avec HNR.

### recherche des paramètres optimaux

Les CH ont plusieurs paramètres répartis dans deux catégories :
- coefficients des termes du node ordering
- les limites qu'on impose au local-search

Ils ont donc cherché les paramètres donnant les meilleurs résultats. Pour cela, ils ont essayé de faire varier les différents paramètres, avec comme objectif de minimiser une fonction, qui existe en deux variantes :
- Agressive variant = minimiser le Query-Time uniquement
- Economical variant = minimiser le produit Query-Time x Preprocessing-Time

Je ne rentre pas dans les détails dans les présentes notes, mais la thèse précise comment ils ont recherché les meilleurs paramètres. Le point intéressant = plus le graphe est gros, plus l'importance relative de l'edge-difference dans les coefficients de l'ordering doit être grosse elle-aussi.

### résultats

Au niveau des résultats non-plus, je ne rentre pas dans les détails.

Voici un chiffre pour fixer les idées : l'upper-bound de la taille des search-space est de l'ordre de quelques centaines de noeuds (800 pour la variante aggressive, 1300 pour la variante économique) pour le graphe Europe, qui comporte pourtant plusieurs dizaines de millions de noeuds !

Note : lorsqu'on doit expand les shortcuts (i.e. si on est intéressé par le plus court-chemin, et non uniquement par sa durée), on consomme plus de mémoire pour stocker le middle-node, et de temps de query pour expand les shortcuts. Concernant la façon dont on stocke le middle-node : il y a compromis à faire entre la taille supplémentaire sur disque (pour le stockage du middle-node) et un léger overhead au preprocess et au query-time.

Une courte section 5.4.6 évoque une customiszation des poids des edges (CCH avant l'heure), permettant d'échanger la métrique travel-time opur la métrique distance, mais les CH ne marchent pas très bien pour ça, vu que l'ordering est trop sensible au changement de métrique. De toutes façons, CCH est un framework plus adapté dans ce cas.

Une autre courte section 5.4.7 montre que le stall-on-demand améliore énormément le temps des queries puisqu'il permet de diviser le temps de query par un facteur entre 2 et 3 !

Il y a une section 5.5 dédiée aux résultats many-to-many. Un point qui semble jouer sur les résultats = le core-size (j'ai pas regardé de très près, c'est pas clair s'il s'agit de la même définition de core que celle utilisée par ULTRA = les noeuds de haut ranks qu'on choisit de ne pas contracter).

Il y a une courte section 5.6 dédiée à l'utilisation des CH pour trouver les transit-nodes de TNR (Transit Node Routing).

In fine, aussi bien pour les requêtes many-to-many que pour TNR, CH est la meilleure base à date.

## Appendix A = Implementation et B = Code Documentation

Dans l'appendix A, il y a une présentation intéressante de l'adjacency-array.

Notamment, ils donnent une façon d'appréhender la structure que j'apprécie : "dans un adjacency-array, on regroupe les edges par source-node".

L'appendix B documente un code auquel je n'ai pas accès, elle est quasi-inutile pour moi.
