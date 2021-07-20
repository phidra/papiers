# (ARTICLE) Customizable Contraction Hierarchies

- **url** = [article](https://arxiv.org/pdf/1402.0402v5.pdf) (md5sum=`aef7c5b64f8a7775dc755596fdf5ecee`, [copie locale](./LOCALCOPIES/1402.0402v5.pdf)). [la thèse](https://i11www.iti.kit.edu/_media/teaching/theses/weak_ch_work-1.pdf) (md5sum=`36e812ef9773bcc96ab0743e6fadee39`, [copie locale](./LOCALCOPIES/weak_ch_work-1.pdf))
- **source** = l'article a été présenté à [SEA 2014](https://di.ku.dk/sea2014/accepted-papers/).
- **auteurs** = Julian DIBBELT, Ben STRASSER and Dorothea WAGNER / Karlsruhe Institute of Technology
- **date de publication** = 2014
- **date de rédaction initiale de ces notes** = août 2020 (possiblement, notes plus anciennes dans le tas)

## Notes synthétiques

Notes à prendre, notamment sur l'article (les notes vrac ci-dessous ont été prises sur la thèse sur WCH et non l'article ; de plus, certaines notes sont globales à CH/dijkstra plutôt que propre à CCH/WCH).

## Notes vrac

Note sur article vs. thèse vs. ... :
- le code fait explicitement référence à la thèse (d'où le fait que la technique soit appelée WCH au lieu de CCH).
- tous les travaux scientifiques postérieurs semblent citer l'article ([exemple1 en 2017](https://arxiv.org/pdf/1504.03812.pdf), [exemple2 en 2019](https://arxiv.org/pdf/1906.11811.pdf), ...)
- je ne l'ai pas lu pour confirmer, mais j'ai vu passer (notamment dans [cet article](https://ad-publications.cs.uni-freiburg.de/GIS_personal_FS_2015.pdf), qui donne une bonne synthèse de CCH et CRP) que [CRP](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/05/crp-sea.pdf) est un concurrent de CCH (permettant également la customization).

Wch permettrait de supporter les turn costs !

La fin de la page 15 (chap 3.1) précise les conditions d'arrêt du dijkstra qui n'étaient pas claires pour moi à la lecture de CH :
- soit la distance minimale de la priority queue du dijkstra est supérieure à la taille du raccourci
- (En effet, tout chemin trouvé par le dijkstra à partir de maintenant sera forcément PLUS GRAND que le raccourci)
- soit tous les successeurs du noeud contracté v sont «settled»
- (en effet, dans ce cas, le dijkstra a trouvé leur distance minimale depuis u, qui ne changera pas : on a déjà trouvé les plus courts chemins qui nous intéressent, inutile de pousser plus loin)

En effet, dans dijkstra, une fois qu'on dépile un noeud de la priority queue, la distance de ce noeud ne sera plus modifiée par la suite :
- le noeud est dit "settled"
- tous les noeuds restants dans la priority queue ont une distance supérieure au noeud en cours ; donc même si on peut atteindre le noeud en cours par un noeud restant, ce sera forcément par un chemin plus long
- inversement, comme le noeud est le premier de la priority queue, ça veut dire que tous les noeuds qui étaient plus proches de nous ont déjà été dépilés ; du coup tous ceux qui auraient pu permettre de l'atteindre d'une autre façon plus rapide ont DÉJÀ été traités
- dit autrement, quand on settle un noeud, on a trouvé le plus court chemin de la source au noeud settled

Minimal contraction hiérarchie = celle où on a poussé le dijkstra jusqu'au bout (on n'aura ajouté de shortcut QUE s'il était nécessaire). En pratique, on arrête le dijkstra local plus tôt, et il se peut qu'on ajoute un shortcut inutile, la contraction n'est pas minimale mais c'est pas bien grave.

Rigolo : trouver un ordering qui minimise le nombre de shortcuts ajoutés a été prouvé comme étant NP-difficile

Visualiser concrètement que CH accélère les queries :
- D'abord, calculer le degré moyen d'un noeud sur le graphe original
- Puis, calculer le degré moyen d'un noeud sur le graphe contracté upward (resp. backward), je m'attends à ce qu'on voie un graphe BEAUCOUP plus petit.
- Et comme dijkstra est en O(E + V.log V), on y gagne
- (Alternative = compter E et V avec et sans CH) 

Witnessless graph contraction :
- On ajoute tous les shortcuts possibles
- (au lieu de n'ajouter que ceux indispensables pour maintenir les plus courts chemins du graphe)
- En fin de contraction, l'overlay  graph est une clique.
- Même witnessless, le résultat de la contraction est toujours TRÈS DÉPENDANT de l'ordering.

Définition de Formal contraction hiérarchie : tous les arcs (réels ou shortcuts) forward (resp. backward) tel que le PCC entre les deux noeuds de l'arc soit l'arc lui même.

Définition de Weak contraction hiérarchie :
- une contraction hiérarchie telle que tout shortcut (i.e. arc n'existant pas dans le graphe original) est issu de la contraction d'un node d'ordre plus petit que les extrémités du shortcut.
- Ndm : c'est obligatoire vu la définition de la contraction hiérarchie!
- Si je comprends bien, «weak» car ne fait pas intervenir la notion de coût/poids ou la notion de plus court chemin pour définir les raccourcis.
- Dit autrement, on se contente d'avoir la contrainte sur "un shortcut est ajouté en contractant les nodes dans l'ordre donné par l'ordering" sans avoir la contrainte "on n'ajoute un shortcut QUE s'il est indispensable pour préserver les plus courts chemins sur le graphe"

Maximal contraction hiérarchie :
- une CH où on ajouté TOUS les shortcuts de chaque noeud contracté.
- Présente des similitudes avec le «elimination game», qui produit un "elimination tree"

Élimination tree :
- pas encore hyper clair, mais la description textuelle sous la définition 3.6 aide :
    + The elimination tree is constructed from the result of the elimination game.
    + For each node it contains only the arc to the lowest higher node.
    + The node with the highest rank is the tree’s root
- Le point important, c'est que tout chemin dans une weak contraction hiérarchie appartient à l'élimination tree du couple graphe+ordering.
- Or, l'elimination game et l'elimination tree qui va avec sont des sujets déjà massivement étudiés dans la littérature.
- Du coup en choisissant un ordering qui minimise la hauteur de l'élimination tree (ce qu'on sait faire avec des heuristiques!), on minimise le chemin dans la contraction hiérarchie.

Nested dissection :
- coupe binaire récursive du graphe.
- Pour un ordering calculé utilisant les nested dissection, la contrainte suivante est toujours vraie = les noeuds de coupe ont un ordre supérieur à tout noeud des deux moitiés coupées.

TODO = tracer la courbe du degré d'un node en fonction de son index d'ordering (pour vérifier visuellement que les derniers noeuds ont une quantité d'edges faramineuse).


Notes vrac sur WCH, issues des divers papiers :
- Notion de lower triangle à creuser.
- Typiquement, pour les Contraction Hierarchies, les nœuds contractés à la fin génèrent beaucoup de shortcuts (à cause des shortcuts déjà créés).
- La contraction converge FORCÉMENT, puisqu'on contracte un set fini de nœuds. Par contre, le temps de convergence peut devenir exponentiel si l'ordering est mauvais.
    + en effet, plus on avance dans la contraction, plus les noeuds seront adjacents à ÉNORMÉMENT d'edges, vu qu'on leur aura ajouté beaucoup de raccourcis
    + du coup, la contraction des noeuds avancés (et la recherche des witness-paths) prendra beaucoup de temps -> le temps de contraction divergera
- Witness path = PCC ne passant pas par le nœud en cours de contraction v, très utile car il "témoigne" qu'il n'y a pas besoin d'ajouter un shortcut pour le triplet (u,v,w)
- Search space :
    + Chaque nœud source s a un graphe G↑(s) constitué des chemins upwards partant de s.
    + Chaque nœud target t a un graphe G↓(t) constitué des chemins downwards arrivant à t.
    + L'intérêt des CH, c'est que ces graphes sont beaucoup plus petits que le graphe G original, et donc que le dijkstra bidirectionnel est beaucoup plus rapide.
    + Le search Space d'un node est le subgraph de G' (G' étant le supergraphe contenant les edges originaux + les shortcuts) accessible depuis z en ne suivant que les arcs upward : G↑


PAS CLAIR : pourquoi faut-il que N soit le noeud d'ordre le moins fort de ses voisins ET DES VOISINS DE SES VOISINS ?
- EDIT : parce qu'on va mettre à jour son cost deux fois, et donc qu'on peut le contracter deux fois
- dit autrement : en quoi le fait de d'impacter deux fois N1 est-il gênant ?
- NOTE : un commentaire dans le code de RE indique que c'est pour éviter de créer deux fois le même raccourci, mais je ne comprends toujours pas...
- l'article Minimum Time-Dependent Travel Times with Contraction Hierarchies donne des indications sur le sujet :

Page 14-15 :

> The gap between the nodes in J is at least three hops instead of two hops, as in the case of I. The reason for this difference is explained at the end of this section in the paragraph about parallelization.

Page 16 : Parallelization.

> Like the contraction of the nodes in a set I as defined by Equation (1), the contraction of the nodes in a set J, as characterized by Property (2), can be performed in parallel quite naturally for shared memory architectures. Also, the simulated contractions of the adjacent nodes can be performed in parallel. But, if simulations are performed in parallel, the following question arises: Is it possible that two threads both simulate the contraction of the same node? This would be redundant work. But this can never happen because of the 3-hop gap between the nodes in an independent node set J as characterized by Property (2).

Du coup j'ai la réponse :
-  il faut recalculer les cost de tous les nodes qui se sont vus ajoutés des edges, qu'on appelle des "impacted" nodes (en effet, dans mon graphe, comme on a retenu N0 comme indépendant et qu'on va le contracter, N1 va se manger un nouveau shortcut)
-  pour calculer le cost d'un node, on simule sa contraction
-  du coup, on va potentiellement simuler la contraction de tout voisin d'un node considéré comme indépendant
-  dans le cas de mon graphe sur papier, si je me suis contenté de prendre des 1-hop independent nodes, N1 est voisin à la fois de N0 et N2
-  son coût va donc être mis à jour deux fois, i.e. on va simuler sa contraction deux fois
-  D'où le commentaire "The goal is to avoid creating the same shortcut twice."

Un "théorème" important (= plutôt un point clé à comprendre), sur l'impact du fait de contracter un node :
- lorsqu'on appelle get_node_cost (qui s'exécute en parallèle), on dit "calcule moi le cost de chaque noeud, EN L'ÉTAT ACTUEL DU GRAPHE"
- d'après le paragraphe ci-dessus, ce cost calculé pour un node N ne changera pas tant qu'il n'y aura pas de nouveaux edges ajoutés/supprimés à N
- en cas de modification du graphe, le cost calculé ne changera QUE pour les noeuds qui auront eu de nouveaux edges ajoutés / supprimés
- de plus, la contraction d'un noeud n'ajoute/supprime des edges QU'AUX NOEUDS voisins d'un node contracté
- dit autrement : le cost calculé n'est modifié (suite à une passe de contraction) QUE pour les noeuds voisins des noeuds contractés
- (ce qui explique qu'on recalcule le cost des nodes voisins des noeuds contractés)
