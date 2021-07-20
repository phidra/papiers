# (ARTICLE) Graph Bisection with Pareto-Optimization

- **url** = [PDF](https://arxiv.org/pdf/1504.03812.pdf) (md5sum=`42910cd624a1b0fff3b749310029a6cd`), [copie locale](./LOCALCOPIES/1504.03812.pdf)
- **source** = conf: [ALENEX 2016](https://epubs.siam.org/doi/book/10.1137/1.9781611974317)
- **auteurs** = Michael HAMANN (Karlsruhe Institute of Technology), Ben STRASSER (Bosch)
- **date de publication** = 2015
- **date de rédaction initiale de ces notes** = octobre 2020

**TL;DR** — l'abstract résume bien l'article :
* We introduce FlowCutter, a novel algorithm to compute a set of edge cuts or node separators that optimize cut size and balance in the Pareto-sense.
* From the computed Pareto-set, we can identify cuts with a particularly good trade-off between cut size and balance that can be used to compute contraction and minimum fill-in orders, which can be used in Customizable Contraction Hierarchies (CCH),
* Our core algorithm runs in `O(c|E|)` time where `E` is the set of edges and `c` is the size of the largest outputted cut
* This makes it well-suited for separating large graphs with small cuts, such as road graphs, which is the primary application motivating our research.
* FlowCutter outperforms the current state-of-the-art both interms of cut sizes and CCH performance
* le code est en accès libre sur [le repo github](https://github.com/kit-algo/flow-cutter) du KIT
* à noter que l'algo a depuis été accéléré en donnant naissance à [InertialFlowCutter](https://arxiv.org/pdf/1906.11811.pdf), qui semble spécifiquement adapté à CCH (combinaison de FlowCutter et de [InertialFlow](https://link.springer.com/chapter/10.1007/978-3-319-20086-6_22))
* de plus, [d'autres algos plus récents](https://drops.dagstuhl.de/opus/volltexte/2019/11173/) existent et semble encore mieux

# Transfert de mes notes brutes

* Deux notions proches quand on parle de partitioning de graphe :
    - graph cut (la frontière est un set d'edges)
    - node separator (la frontière est un set de nodes)
* Flow cutter (2016) = algo utilisé pour trouver un ordering
* Preprocessing nécessite de  calculer un node separator à peu près équilibré, mais pour CH, ce critère d'équilibrage est MOINS important que le crtière visant à minimiser la taille du cut.
* Un papier qui semble intéressant : [graph partitioning with natural cuts](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/12/punchTR.pdf) en 2010
* Bisection = partition en deux groupes.
* Il existe beaucoup de problèmes liés au partitioning de graphes :
    - minimiser la somme des poids des arêtes sur la coupe
    - minimiser l'imbalance
    - minimiser le nombre d'arêtes sur la coupe.
* Bisection = bicriteria problem minimisant la cutsize et l'imbalance .
* L'article essaye de renvoyer un pareto-front, alors que les approches classiques minimisent la cutsize, en ayant comme contrainte que l'imbalance ne dépasse pas x%
    - a priori, c'est encore un problème un peu différent (et plus facile) que de trouver le partitioning de cutsize minimale, pour une imbalance donnée
    - plus précisément, avec ma compréhension des choses :
        + problème 1 = trouver la cutsize minimale conduisant à une imbalance inférieure à X%
        + problème 2 = étant donné une imbalance d'exactement X%, trouver la cutsize minimale
* Notion importante (pour WCH) (peut-être à mettre en lien avec les lower triangle ?), C'est que l'auxiliary data de CCH est un supergraphe chordal du road graph.
* Graphe cordal est aussi appelé graphe triangulé : chaque cycle est un triangle.
* Nested dissection
* Une partition peut être assimilée à son set de cut-arcs
* Un optimum de pareto est un "point" sur le front de pareto
* Trouver une coupe minimale (en terme de cutsize) est un problème P-difficile, donc a priori solvable efficacement.
* En revanche, la plupart des problèmes couplant la cutsize à l'imbalance sont NP-difficiles
* La section 3.3 du papier sur FlowCutter donne un excellent overview de CCH . Notamment, les 3 critères suivants sont pertinents :
    - smaller search Space (pour accélérer les queries)
    - fewer triangles in G' (pour accélérer la customisation)
    - fewer edges in G' (pour diminuer la consommation mémoire)
* Le papier donne également une Overview du lien entre le partitioning et l'ordering
* La section 3.4 donne un résumé de la théorie des tree decompositions, et donne des liens vers des surveys
* Glossaire : ce qu'on appelle l'index d'ordering est plutôt appelé "rank" dans la littérature.
* Perfect elimination ordering
* ----------------------------------------
* La fin de la section 3.5 donne un rapide overview des nested dissection.
* Si un jour j'ai le temps de m'intéresser aux maths derrière le partitionnement du graphe (qui font que WCH marche bien), la section 3 de ce papier sera précieuse.
* Note : métis propose ndmetis, spécialement fait pour faire du node-ordering
* D'après la comparaison avec metis, sur l'Europe, FlowCutter divise par trois le temps de customisation, divise par trois le temps de query, au prix d'une multiplication par 20 (au moins, avec f3) du temps d'ordering.
* ----------------------------------------
* Nombre de triangles corrélés à la rapidité de la customisation.
* Nombre d'arcs dans la CH corrélés à la rapidité de la query.
* Métis semble plutôt mauvais sur la taille du search Space -> amélioration possible des performances à la query
* ----------------------------------------
* Autre amélioration potentielle : métis produit plus de shortcuts que flowcutter
* En revanche, metis est très, très, très rapide.
* Page 23, ils mentionnent un bug empêchant métis de trouver le Mississippi
* L'ordre de contraction des nodes pour WCH semble être la conjonction 1. d'un partitioner (e.g. metis) et 2. d'une nested-dissection strategy qui applique récursivement le partitioner, et fournit l'ordering.
* Au passage, le nd de ndmetis est sans doute là pour nested dissection (à vérifier), donc on retrouve cette conjonction dans le nom de l'exécutable
* Papier : search Space Size in contraction hierarchies
* Référence vers un papier qui utilise ch pour faire du calcul d'iti multimodal = user-constrained multimodal route planning
* Sur le papier lui-même : de précédents travaux ont montré que sous réserve d'une contrainte sur la highway dimension du graphe soit vraie (conjecturée vraie pour les road networks), alors les ch ont un gain prouvé. Le présent papier prouve des résultats en n'utilisant que la topologie du graphe, et pas sa métrique.
* Le papier semble prouver que la nested dissection produit des orderings intéressants.
* Résumé très succint des ch et de Dijkstra
* Notion de search space vs reverse search space
* Point intéressant, ils s'intéressent à un Dijkstra sans halting criterion, vu que c'est le worst case scénario qui les intéresse.
