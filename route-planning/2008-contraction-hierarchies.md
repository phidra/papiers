# (ARTICLE) Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks

- **url** = [article simplifié](http://algo2.iti.kit.edu/schultes/hwy/contract.pdf) (md5sum=`9dfe56278407459bb0e2d9a781bcf8f4`, [copie locale](./LOCALCOPIES/contract.pdf)) / [thèse](https://algo2.iti.kit.edu/documents/routeplanning/geisberger_dipl.pdf) (md5sum=`b31efbba33a3bc527523eed6b8722a2f`, [copie locale](./LOCALCOPIES/geisberger_dipl.pdf))
- **source** = [WEA 2008](https://dl.acm.org/doi/proceedings/10.5555/1788888)
- **auteurs** = Robert GEISBERGER (dont c'est la thèse), Peter SANDERS, Dominik SCHULTES, and Daniel DELLING / KIT
- **date de publication** = 2008
- **date de rédaction initiale de ces notes** = juin 2020 (possiblement, notes plus anciennes dans le tas)


## Notes synthétiques

Le travail de synthèse reste à faire : je me suis contenté de déplacer mes notes vrac de l'époque à cet emplacement.

----

* [(ARTICLE) Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks](#article-contraction-hierarchies-faster-and-simpler-hierarchical-routing-in-road-networks)
   * [Notes synthétiques](#notes-synthétiques)
   * [Notes spécifiques sur les critères d'ordering (notes prises en 2021)](#notes-spécifiques-sur-les-critères-dordering-notes-prises-en-2021)
      * [TL;DR](#tldr)
      * [Notes détaillées](#notes-détaillées)
         * [principe général](#principe-général)
         * [catégorie 1 = Edge difference](#catégorie-1--edge-difference)
         * [catégorie 2 = Cost of Contraction](#catégorie-2--cost-of-contraction)
         * [catégorie 3 = Uniformity](#catégorie-3--uniformity)
            * [Contracted neighbors](#contracted-neighbors)
            * [Sum of original edges of the new shortcuts / Hop-quotient](#sum-of-original-edges-of-the-new-shortcuts--hop-quotient)
            * [Voronoï region](#voronoï-region)
         * [catégorie 4 = Cost of Queries](#catégorie-4--cost-of-queries)
            * [Compréhension avec les mains](#compréhension-avec-les-mains)
            * [Illustration avec la "dead-end valley"](#illustration-avec-la-dead-end-valley)
            * [Implémentation du level](#implémentation-du-level)
            * [En quoi contracter les nodes à fort level plus tard nous aide ?](#en-quoi-contracter-les-nodes-à-fort-level-plus-tard-nous-aide-)
         * [catégorie 5 = Global measures](#catégorie-5--global-measures)
   * [Notes vrac prises en 2020 ou avant](#notes-vrac-prises-en-2020-ou-avant)
      * [NdM](#ndm)
      * [Copie de mes notes manuscrites (à trier puis merger avec les autres notes)](#copie-de-mes-notes-manuscrites-à-trier-puis-merger-avec-les-autres-notes)
      * [Chapitre 2 = Contraction :](#chapitre-2--contraction-)
      * [Chapitre 3 = Node-Ordering :](#chapitre-3--node-ordering-)
      * [Chapitre 4 = Query](#chapitre-4--query)
      * [Chapitre 6 = Experiments](#chapitre-6--experiments)
      * [Chapitre 7 = Conclusions](#chapitre-7--conclusions)
      * [Page 3 - chapitre 2 "contraction"](#page-3---chapitre-2-contraction)
         * [Ma compréhension des notations](#ma-compréhension-des-notations)


## Notes spécifiques sur les critères d'ordering (notes prises en 2021)

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

### Notes détaillées

La partie de la thèse analysée est le sous-chapitre **3.2 Node Order Selection** (du chapitre 3 = Cotnraction Hierarchies), entre les pages 14 et 22 incluses.

#### principe général

> We will first present a simple and extensible heuristic using a priority queue and then present several possible priority terms partitioned into five categories.
>
> We can chose any node order to get a correct procedure. However, this choice has a huge influence on preprocessing and query performance.
>
> The priority of a node u is the linear combination of several priority terms and estimates the attractiveness to contract this node

Le principe général est de contracter séquentiellement les noeuds, le prochain noeud à ordonner étant à chaque étape celui qui minimise (grâce à une priority-queue) un score d'ordering.

**Important** : la qualité de l'ordering a une forte importance sur la qualité de la CH résultante.

*Section 3.2.1  Lazy Updates* = explications sur un détail d'implémentation pour faire l'update du score des nodes (notamment, la *dégradation* du score d'un node) le plus tard possible.

#### catégorie 1 = Edge difference

**OBJECTIF** = la CH résultante reste sparse, i.e. ne contient que peu d'edges.

*Section 3.2.2  Edge Difference* :

> Arguably the most important priority term is the edge difference. Intuitively, the number of edges in the remaining graph should decrease with the number of nodes.

Edge difference = (nombre d'edges dans le graphe APRÈS contraction de U) - (nombre d'edges dans le graphe AVANT contraction de U)

[La thèse TCH](./2014-TDCH-thesis.md), au sous-chapitre 5.2.3 Ordering the nodes, indique qu'il vaut mieux prendre l'edge-quotient (comme fait RoutingKit) plutôt que l'edge-difference :

> Note that the edge quotient works better than the more intuitive term edge difference would do. This is because the values of the difference could get so large that other terms would not have enough influence any more.

À noter qu'un critère dérivé semble être *New edges* = compter les nouveaux edges ajoutés au graphe par la contraction de U (mais d'après le papier, ce critère n'est pas intéressant, et ils l'ont ignoré).

NOTE : j'ai pas creusé, mais on dirait que (en théorie), l'edge difference de TOUS les nodes (et pas uniquement des voisins de U) peuvent être affectés par la contraction de U.
En théorie, après contraction de U, il faudrait recalculer le score d'ordering de tous les nodes du graphe. En pratique, ne recalculer que les voisins semble être une approximation acceptable.

#### catégorie 2 = Cost of Contraction

**OBJECTIF** = le temps nécessaire au preprocessing reste faible (ou en tout cas ne diverge pas).

*3.2.3  Cost of Contraction* :

> Those local searches to find witness paths have the biggest share of the preprocessingtime.

C'est la recherche des witness-path qui prend le plus de temps de preprocessing.

> That is the reason why we chose the sum of the search space sizes of the local Dijkstra searches for the cost of contraction for a node u, more precisely the number of settled nodes

L'idée est de dire : "si un node est lourd à contracter, on essaye de le contracter plus tardivement". Ma compréhension des choses, c'est qu'en le contractant plus tard, il y aura moins de nodes dans le graphe de contraction (vu qu'on sera plus haut dans la hiérarchie, on aura entretemps **retiré** des nodes du graphe), donc le search-space sera plus petit.

Note d'implémentation : c'est assez galère d'updater les scores des autres nodes lorsqu'on contracte un node U, puisque U peut apparaître dans le search-space size de beaucoup d'autres nodes... En pratique, donc, ils se contentent d'updater la taille du search-space-size des voisins de U.

#### catégorie 3 = Uniformity

**OBJECTIF** = répartir la contraction des nodes sur tout le graphe = éviter que des nodes proches soient contractés successivement.

*3.2.4  Uniformity* :

L'idée derrière cette catégorie de critères est que si on a un chemin linéaire (une "dead-end valley"), il faut éviter de contracter les noeuds du chemin linéaire en séquence consécutive. En effet, si contracte les noeuds consécutivement, la forward-propagation va avancer d'un noeud à la fois sur le chemin linéaire, au lieu de sauter plusieurs noeuds d'un coup grâce à un shortcut. L'idée derrière est qu'en contractant plus tardivement les nodes dont des voisins ont déjà été contractés, on tend à une profondeur de search-tree logarithmique plus tard, au query-time.

NdM = avec mes yeux naïfs, ce critère d'uniformité semble se rapprocher pas mal de la notion de hierarchy-depth, puisque dans les deux cas, on cherche à ce que les shortcuts créés "sautent" plusieurs noeuds d'un coup... Je pense que la différence est dans le moyen d'y parvenir : les critères d'uniformité sont un moyen indirect à l'ordering-time, là où la hierarchy-depth est un moyen plus directement lié à ce qui se passe au query-time (mais peut-être également plus approximé car on travaille avec l'upper-bound de la profondeur du shortest-path-tree)

##### Contracted neighbors

> Contracted neighbors = count for each node the number of previous neighbors that have been contracted.

Premier critère de cette catégorie = contracted neighbours (c'est assez logique : si on juge moins prioritaire les noeuds dont les voisins ont déjà été contractés, l'ordering va naturellement se répartir sur le graphe).

##### Sum of original edges of the new shortcuts / Hop-quotient

> Count the number of original edges of the newly added shortcuts during the contraction of a node
>
> Related to the contracted neighbors counter is another priority term, it also counts the number of already contracted nodes, but now on edges. To get a better intuition for this priority term, we can also say that we count the number of edges in the original graph a shortcut represents.

Ma compréhension : chercehr à minimiser le nombre de hops réels que représente un shortcut, c'est surtout un moyen de s'assurer mécaniquement de répartir la contraction sur tous les nodes du graphe (p.ex. on ne contractera un node U proche d'un node V déjà contracté *QUE* s'il ne reste plus de nodes à contracter sans voisins déjà contractés).

Ce critère m'intéresse beaucoup, car il est très proche du hop-quotient implémenté dans RoutingKit ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L563) : `(1000*added_hop_count) / removed_hop_count`).

La différence entre les deux : l'implémentation de RoutingKit ne se contente pas de minimiser le **nombre** de hops réels qu'un shortcut représente, mais plutôt le **quotient** du nombre de hops ajoutés sur le nombre de hops removed. En pratique, ça veut dire que à nombre de hops ajouté égal, on cherche à favoriser les contractions qui dégagent (par exemple par le biais de witness-path montrant que les edges ne sont pas nécessaires) les shortcuts avec beaucoup de hops réels.

Mineur : le papier parle également d'une autre motivation liée à transit-node routing (qui ne m'intéresse pas vraiment) : 

> This property should avoid shortcuts that represent too many original edges. The motivation is to use it for transit-node routing where the current implementation [35] only supportsfew node levels

Enfin, un autre avantage mentionné en passant est que si on diminue le hop-count des shortcuts, leur unpacking sera plus rapide.

##### Voronoï region

Je me suis contenté de survoler, mais en gros, la voronoi-region d'un node pas encore contracté augmente au fur et à mesure qu'on contracte ses voisins.  Du coup, il faut chercher à contracter PLUS TARDIVEMENT les nodes dont la voronoi region est la plus grande. Ça assurera qu'on répartit la contraction des nodes, et qu'on évitera de contracter des séquences linéaires de nodes.

#### catégorie 4 = Cost of Queries

**OBJECTIF** = garantir qu'au query-time, la profondeur des shortest-path-tree reste faible.

*3.2.5  Cost of Queries* :

Préambule / rappel = au query-time, la forward-propagation (resp. backward) fait grossir petit à petit un arbre des plus courts chemins à partir de la source, jusqu'à trouver le meeting-node entre forward et backward, qui a le plus haut rank du chemin final (en réalité, le critère d'arrêt d'un dijkstra bidirectionnel est un peu plus compliqué).

Ce critère vise à garantir que vu l'ordering des node choisi, la profondeur du shortest-path-tree dans le pire cas reste limitée, et donc garantir des query-time rapides.

Ma compréhension des choses, c'est que ce critère est intimement lié au critère d'uniformité. C'est d'ailleurs plutôt confirmé par cette phrase :

> The uniformity properties of the previous section are useful to speed up the queries in the resulting contraction hierarchy

Et l'ajout d'un critère explicite sur la taille des search-tree va dans le même sens :

> Adding a property that tries to limit the size of the querysearch space results in even faster query time

##### Compréhension avec les mains

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

##### Illustration avec la "dead-end valley"

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

##### Implémentation du level

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

##### En quoi contracter les nodes à fort level plus tard nous aide ?

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

#### catégorie 5 = Global measures

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

## Notes vrac prises en 2020 ou avant

### NdM

NdM : quand on cherche à contracter un noeud v
- on va supprimer le noeud v du graphe
- du coup, il faut ajouter des raccourcis partout où v est sur un plus court chemin (PCC)
- si le plus court chemin entre le noeud u et le noeud w passe par v, il faut rajouter un raccourci de (u;w)
- moi : inutile de regarder pour tous les noeuds du graphe : il suffit de regarder s'il y a des plus courts chemins entre deux noeuds VOISINS de v !
- (en effet, si un PCC de u à w passe par v, alors il passera forcément par deux voisins de v)
- le hic : comme à ce stade on travaille sur l'overlay graph (auquel il manque des noeuds, mais qui contient des edges supplémentaires) alors les "voisins" de v peuvent être très TRÈS nombreux, et très éloigné les uns des autres !
- il peut donc être compliqué de rechercher les plus courts chemins entre tous les voisins de v :-(

NdM = pourquoi CH accélère le routing ?
- parce qu'on ne peut aller QUE vers des noeuds d'ordre supérieur
- à chaque nouveau noeud relaxé dans un dijkstra unidirectionnel (peu importe le sens), on restreint le nombre de noeud qu'on peut explorer
- on les restreint même de plus en plus au fur et à mesure qu'on avance dans l'algo :
    + supposons que le graphe ait 10000 nodes
    + alors lorsqu'un node relaxé a l'order 9500, il ne nous reste plus que 500  (au lieu de 10000) noeuds possibles à explorer

NdM = pourquoi est-ce important de répartir la contraction uniformément ?
- car (en reprenant l'exemple juste au dessus), les 500 nodes restants sont tous dans mon voisinage, il faudra que je les explore tous
- si à l'inverse ils sont répartis uniformément sur le graphe et que du coup je n'en ai que 20 dans mon voisinage, alors je n'aurais que 20 noeuds (au lieu de 500) à explorer pour terminer l'exploration de cette branche du graphe
- Dit autrement : la répartition des noeuds du graphe permet d'élaguer rapidement des branches du graphe qu'on n'explorera pas.
- Une autre façon de voir les choses :
    + le meeting-node où se rejoignent les deux dijkstra unidirectionnels sera le node d'ordre le plus élevé sur le chemin
    + si l'ordering a regroupé TOUS les nodes d'ordre élevés à un endroit (par exemple au nord)
    + alors on est sûr que le meeting point se trouvera au nord
    + et si on veut faire un trajet du sud au nord, le dijkstra qui part du sud devra explorer toute la France pour trouver son meeting-point au nord

Après seconde lecture de l'article, ce qui ne reste pas clair :
- différence entre limiter le searchSpaceSize et limiter le nombre de hops
- implémentation du dijkstra pour choisir si on ajoute un shortcut, notamment comment utiliser la limite max
- le stall-on-demand

Sans doute que la lecture de la thèse aiderait...

Au sujet de la difficulté à trouver un bon ordering, la [thèse WCH](https://i11www.iti.kit.edu/_media/teaching/theses/weak_ch_work-1.pdf) donne des références sur le sujet :
- But finding an order which minimizes the amount of added shortcuts is known to be NP-hard [BCK+10]
    + Reinhard Bauer, Tobias Columbus, Bastian Katz, Marcus Krug, and DorotheaWagner.
    + Preprocessing Speed-Up Techniques is Hard.
    + In Proceedings of the 7thConference on Algorithms and Complexity (CIAC’10), volume 6078 ofLectureNotes in Computer Science, pages 359–370. Springer, 2010.

Par ailleurs, concernant le preuve que CH accélère les queries en limitant le search space size, la même thèse donne des références :
- In [BCRW13,Col12], a theoretical framework to study Contraction Hierarchies was developed. It enables proving upper bounds on the search space size for certain classes of graphs
    + Reinhard Bauer, Tobias Columbus, Ignaz Rutter, and Dorothea Wagner.
    + Search-Space Size in Contraction Hierarchies.
    + In Proceedings of the 40th Inter-national Colloquium on Automata, Languages, and Programming (ICALP’13),volume 7965 ofLecture Notes in Computer Science, pages 93–104. Springer,2013.

### Copie de mes notes manuscrites (à trier puis merger avec les autres notes)

Heuristique de contraction :
- edge difference
- spread = contract nodes uniformely

bidirectional dijkstra :
- forward dans G↑
- backward dans G↓
- meeting node = le noeud qui a l'ordre le plus élevé du path trouvé à la query
- (NdM : d'où l'importance de spread uniformément la contraction : si tous les noeuds d'ordre élevés sont au même endroit, ça va pas aller)


### Chapitre 2 = Contraction :
- "witness" path = un chemin qui témoigne (i.e. qui "prouve") qu'il est inutile d'ajouter le shortcut pour le triplet (u,v,w) considéré
- (en d'autres termes, un witnesse path est un plus court chemin de u à w qui ne passe pas par v)
- pas clair : "hop" ? Semble être un moyen de limiter le coût du dijkstra local = de la vérification de si oui ou non on ajoute un shortcut. hop est adapté dynamiquement, pas clair comment.

### Chapitre 3 = Node-Ordering :

L'ordering utilise une priority queue sur une combinaison linéaire de différentes facteurs.

Quand le node v est contracté, ça affecte la priorité d'autres nodes :-/

Pour mitiger :
- lazy updates
- recalcul prio des voisins de v
- recalcule régulièrement TOUTES les priorités

La combinaison linéaire pour définir les priorités utilise :
- edge difference
- uniformité pour spread la contraction un peu partout (QUESTION : maximal-independent set = ??)
- deleted neighbours = voisins déjà contractés
- voronoi regions :
    + VR(v) = { u∈V ; u est plus proche de v que de n'importe quel autre noeud }
    + on utilise sqrt(size(VR)) dans la priority function et on contracte les noeuds à petite VR d'abord
- optionnel = cost of contraction
- optionnel = cost of queries (pas clair)
- optionnel = global measure (NdM = FRC7 ?)

### Chapitre 4 = Query

Stall-on-demand = pas clair

EDIT : dans un autre papier sur TCH, il y a une mention qui peut m'aider à comprendre :

> During the bidirectional search we perform stall-on-demand [4, 2]: The search stops at nodes when we already found a better route coming from a higher level.

Dans [ce papier](http://algo2.iti.kit.edu/documents/routeplanning/tch_alenex09.pdf), j'ai :

> The stall-on-demand technique identifies such nodes w by checking whether(downward) edges coming into w from more important nodes give shorter paths than the (upward) Dijkstra search.

Il y a un chapitre très détaillé sur le Stall-on-demand à la page 219 de [la thèse sur TCH](http://digbib.ubka.uni-karlsruhe.de/volltexte/documents/3569195)

### Chapitre 6 = Experiments

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

### Chapitre 7 = Conclusions

Goal-directed technique = algos orientant la recherche de la solution "vers" la cible (e.g. A*)

Il y a également une définition et quelques exemples de techniques) dans [ce papier](https://publikationen.bibliothek.kit.edu/1000014952) :

> Goal-Directed Approaches direct the search towards the target t by preferring edges that shorten the distance to t and by excluding edges that cannot possibly belong to a shortest path to t. Such decisions are usually made by relying on preprocessed data.


### Page 3 - chapitre 2 "contraction"

#### Ma compréhension des notations

- d(u,w) = distance(u,w) = coût du plus court chemin de u à w. (u,w) n'est pas forcément un edge
- c(u,w) = cost(u,w) = poids de l'edge (u,w). (u,w) est forcément un edge.
- Même si (u,w) est un edge, on n'a PAS FORCÉMENT d(u,w) == c(u,w) :
    + sur le graphe original G, il existe peut-être un AUTRE chemin que l'edge (u,w) permettant d'aller de u à w, de façon plus courte
    + sur l'overlay graph G′, si (u,w) est un raccourci, il existe peut-être un autre  chemin que le raccourci pour aller de u à w, de façon plus courte

> Recall from the introduction that when contracting node v, we are dealing with an overlay graph G′=(V′,E′) with V′=v..n and an edge set E′ that preserves shortest path distances wrt the input graph.

Au moment où on contracte le noeud v, l'overlay graph G' :
- contient les noeuds d'ordre v à n (les autres noeuds ont déjà été contractés, donc ne sont plus dans l'overlay graph)
- contient les edges originaux, ainsi que les raccourcis créés lorsqu'on a contractés les noeuds de 1 à (v-1)

> In G′, we face the following many-to-many shortest-path problem: For each source node u ∈ v+1..n with (u,v) ∈ E′and each target node w ∈ v+1..n with (v,w) ∈ E′, we want to compare the shortest-path distance d(u,w) with the shortcut length c(u,v) + c(v,w) in order to decide whether the shortcut is really needed

- considérant deux NOEUDS u et w VOISINS de v tel que u → v → w soit un chemin (en effet, (u,v) ∈ G′ et (v,w) ∈ G′)
- (dit autrement, on s'intéresse à tous les PRÉDÉCESSEURS u de v, et tous les SUCCESSEURS w de v)
- on veut calculer la longueur d(u,w) du plus court chemin PCC(u,w), et la comparer avec c(u,v) + c(v,w) = la longueur du raccourci qu'on ajouterait en supprimant v
- l'idée est de NE PAS ajouter le raccourci (u,w) si c(u,v) + c(v,w) > d(u,w), car ça voudra dire que le PCC(u,w) NE PASSE PAS par v (mais "contourne" v)

> A simple way to implement this is to perform a forward shortest-path search in the current overlay graph G′ from each source, ignoring node v, until all targets have been found

- pour CHAQUE prédécesseur u de v, on calcule un dijkstra pour calculer le plus court chemin de u à N'IMPORTE QUEL noeud de G′
- (en effet, dijkstra peut calculer le plus court chemin d'une source à TOUS les sommets du graphe)
- (note : comme on est dans l'overlay graph, les prédécesseurs de v incluent les prédécesseurs via des raccourcis -> il peut y en avoir beaucoup, et très éloignés de v !)
- ainsi, on calculera toute les distances de u à tous les noeuds de G′ et notamment à tous les w, donc on aura calculé d(u,w), ce qui nous intéresse

> We can also stop the search from u when it has reached distance d(u,v) + max{c(v,w) : (v,w) ∈ E′}
- lorsqu'on cherche le PCC de u à w (mais ne passant pas par v), même si on en trouve un...
- ...si le PCC trouvé a un coût supérieur à d(u,v) + max{cost(v, w)}...
- ... alors ce PCC NE NOUS SERVIRA PAS à exclure le raccourci passant par v (car on est alors sûr que le PCC trouvé sera plus LONG que le raccourci)

> Our actual implementation uses a simple, asymmetric form of bidirectional search inspired by [10]: For each target node w we perform a single-hop backward search. For each edge (x,w) ∈ E′ we store a bucket entry (c(x,w), w) with node x. 

- pour tous les PRÉDÉCESSEURS de w, on stocke une entrée dans un bucket (hashtable?) avec le coût de l'edge du prédécesseur à w

> This way, forward search from u can be limited to distance :

```
c(u,v)+ max{c(v,w)} − min{c(x,w)}
        w:(v,w)∈E′    x:(x,w)∈E′
```
L'objectif semble être de prune au plus vite le dijkstra pour pas qu'il nous coûte trop cher.  On semble additionner :
- le coût de l'edge d'entrée (u,v)
- le coût MAXIMAL de l'edge de sortie (v,w)
Auquel on retranche : le coût du plus petit prédécesseur de w (QUESTIOn : même s'il n'est pas un successeur de u ?).

QUESTION : je ne comprends pas bien pourquoi on peut limiter le coût max du dijkstra à ça ?

- Je suppose en préambule qu'il s'agit d'être sûr que tous les PCC au delà de cette valeur seraient plus LONG que c(u,v) + c(v,w)
- ça revient donc à dire qu'en majorant  max{c(v,w)} − min{c(x,w)}  ,  on est sûr de majorer  c(v,w)  :
    + max{c(v,w)} − min{c(x,w)}   est FORCÉMENT supérieur à   c(v,w)
- Je vois 4 cas :
    1.  (v,w) a le coût le plus GRAND des successeurs de v  ET  (v,w) a le coût le plus PETIT des prédécesseurs de w
        si v est le plus petit des successeurs de w, par définition, (v,w) est le moyen le plus rapide de relier v à w
    2.  (v,w) a le coût le plus GRAND des successeurs de v  ET  (v,w) a le coût le plus GRAND des prédécesseurs de w
    3.  (v,w) a le coût le plus PETIT des successeurs de v  ET  (v,w) a le coût le plus PETIT des prédécesseurs de w
    4.  (v,w) a le coût le plus PETIT des successeurs de v  ET  (v,w) a le coût le plus GRAND des prédécesseurs de w

Mmmmh, non, décidément, c'est pas clair...

> When reaching a node x, we scan its bucket entries. For each entry (C,w), we can infer that there is a path from u to w of length d(u,x)+C.

- pour chaque node x atteint dans le dijktra, on sait qu'on peut atteindre w en d(u,x) + c(x,w)
- ce dernier terme c(x,w) est déjà dans la bucket-list

> Since exact shortest path search for contraction can be rather expensive, we have implemented two ways to limit the range of searches: We can limit the number of hops (edges) used in any path〈u,...,w〉, and we can limit the total search space size of a forward search

- mon interprétation de "limiter le total search space size" = on limite le nombre de NOEUDS possibles dans la recherche locale du plus court chemin (on arrête d'explorer quand on a exploré Nmax noeuds)
- mon interprétation de "limiter le nombre de hops" = on limite le nombre d'EDGES possibles dans un plus court chemin (on arrête d'explorer quand un chemin contient Emax edges)
- QUESTION = mais tel que l'article le présente, limiter le nombre de hops = limiter le nombre d'edges dans le chemin ? Du coup quelle différence avec limiter le search space ?

Apparemment, leur implémentation "hop" s'inspire [de ça](http://algo2.iti.kit.edu/documents/routeplanning/distTable.pdf) (à regarder ?)
