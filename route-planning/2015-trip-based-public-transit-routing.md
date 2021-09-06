# (ARTICLE) Trip-Based Public Transit Routing

- **url** = [lien](https://arxiv.org/abs/1504.07149), [PDF](https://arxiv.org/pdf/1504.07149.pdf) (md5sum=`34f635be34a406929c725f912ddaef0e`), [copie locale](./LOCALCOPIES/1504.07149.pdf)
- **source** = conf: [ALGO/ESA 2015](http://algo2015.upatras.gr/esa/accepted.html), [proceedings](https://link.springer.com/book/10.1007%2F978-3-662-48350-3)
- **auteurs** = [Sascha WITT](https://algo2.iti.kit.edu/english/2373.php) (Karlsruhe Institute of Technology)
- **date de publication** = 2015
- **date de rédaction initiale de ces notes** = septembre 2021

**TL;DR** :

* le TL;DR reste à faire

## Transfert de mes notes brutes

section 3.1 : parmi tous les transfers trip-à-trip possibles, seuls quelques uns contribuent aux plus courts chemins

objectif du preprocess : identifier ces transfers trip-à-trip qui sont utiles

**NOTE IMPORTANTE :**

- un "trip" est un voyage physique dans un véhicule en particulier
- il est donc fixé dans le temps !
- ainsi, parler du "stop p(exposant i)(indice t) = le i-ième stop du trip `t` revient à définir sans ambigüité un StopEvent
- (i.e. quand on mentionne un stop d'un trip on définit exactement à quelle heure on arrive au stop, et à quelle heure on en repart)

## STEP 1 = initial computation :

**OUTPUT** = un set de transfers (trip→trip) possible

### algo :

- on itère sur tous les trips
- pour chaque trip, on itère sur tous ses stops
- pour chaque stop (source du potentiel transfer), on regarde quels sont les stops (target) accessibles à pied depuis le stop source
- pour chacun de ces target-stops, on itère sur toutes les lignes `L` qui l'empruntent
- pour chacune de ces lignes, on recherche le premier de ses trips qui soit matériellement empruntable
- (i.e. compte-tenu du temps de marche à pied, on a le temps d'emprunter le target-trip)
- on mémorise le transfert correspondant (et tous les trips de `L` ultérieurs peuvent être ignorés)

### Précisions / optimisations :

- inutile de calculer les transfers DEPUIS le PREMIER stop d'un trip
- en effet, emprunter ces transfers revient à ne pas utiliser le trip du tout
- (vu qu'on fait un transfert avant même de prendre le trip)
- symétriquement, inutile de calculer les transferts VERS le DERNIER stop d'un trip
- par ailleurs, à part pour "revenir en arrière", les transferts entre deux trips d'une même ligne sont inutiles
- (NdM : j'ai l'impression qu'ils sont inutiles tout court : ces retours en arrière pourraient être remplacé par de l'attente à un stop ultérieur ?)

## STEP 2 = reduction

**objectif** = supprimer les transfers "inutiles", i.e. qui ne contribuent à aucun plus court chemin d'aucun front de Pareto renvoyé.

### cas 1 = U-turn transfers

* en gros, il s'agit des transfers ou le target-trip nous ramène à un stop antérieur du source-trip
* ces transfers ne sont jamais utiles, sauf dans le edge-case où il est plus rapide de faire le transfer que de marcher
* dit autrement : marcher pour rejoindre le target-trip + utiliser le target-trip permet de rejoindre un stop du source-trip plus rapidement qu'en restant dans le source-trip, au point qu'on louperait un trip ultérieur à cause de cette lenteur (et qu'il est donc indispensable de conserver le U-turn transfer pour ne pas louper le trip ultérieur)
* ça peut arriver si le temps de marche à pied depuis le stop antérieur du source-trip est très très grand (et que le transfer+target-trip sont plus rapides)

### cas 2 = transfer reduction

FIXME : je ne poursuis pas plus l'analyse de la réduction, j'en reste au fait que le résultat du preprocessing est un set des transferts utiles entre le i-ième stop d'un trip `t` et le j-ième stop d'un trip `u`.

## Earliest Arrival Query

### FIXME : titre et découpage à revoir

INPUT = heure de départ `τ`, stop source `p_src`, stop target `p_tgt`

DONNÉES PRÉPROCESSÉES = un set des transferts utiles entre le i-ième stop d'un trip `t` et le j-ième stop d'un trip `u`.

OUTPUT = un front de Pareto optimisant l'heure d'arrivée et le nombre de trajets TC (pour chaque nombre de transfert, on veut l'heure d'arrivée la plus tôt).

**FIXME = notes vrac en cours d'analyse du papier**

> During the algorithm, we remember which parts of each trip `t` have already been processed by maintaining the index `R(t)` of the first reached stop, initialized to `R(t) ← ∞` for all trips.

Pas encore clair ce qu'est cet index...

D'après la définition, il existe un index `R` pour chaque trip `t`. Pour un trip donné, l'index stocke un stop (ou plus exactement son index) = l'index du premier stop du trip atteint (initialisé à `∞` = on n'atteint aucun stop du trip).

> We also use a number of queues `Qn` of trip segments reached after `n` transfers 

Idem, pas encore clair...

> We also use [...] a set `L` of tuples `(L, i, ∆τ)` [...] The latter indicates lines reaching the target stop ptgt

Ici, on dirait qu'il s'agit des lignes capables de rejoindre le stop target`p_tgt`, on retient :

- toutes les lignes passant directement par le stop `p_tgt`
- toutes les lignes passant indirectement par `p_tgt` (i.e. elles passent par un autre stop `q` pour lequel il existe un footpath permettant de rejoidnre `p_tgt` depuis `q`).

Le `Δτ` dans la formule semble représenter le temps nécessaire depuis le i-ième stop de la ligne `L` pour rejoindre le stop `p` (il est égal à 0 si `p` est un stop de `L`, et égal au temps de marche à pied entre `q` et `p` sinon).

En résumé, ce set `L` indique les différentes façons possibles de rejoindre le stop `p`.

----

> We start by identifying the trips travelers can reach from `p_src` at time `τ`

Plutôt straightforward : on regarde les TRIPS (d'une façon générale, tout l'algo semble basé sur les trips) accessible depuis le stop source à l'heure de départ, en incluant ceux qu'on peut attraper via un footpath.

Pour cela, on itère sur les lignes qui passent par {le stop ou l'un de ceux accessibles via un footpath}, collectivement désignés par `q` dans l'article.

Pour chaque ligne passant par l'un de ces stops `q`, on garde le **premier** trip (i.e. le trip le plus tôt de la ligne).

Bien sûr, on ne s'intéresse qu'à ceux qu'on peut réellement attraper, i.e. ceux qui partent après `τ` (ou après `τ + temps de marche à pieds` si `q` est un stop accessible par un footpath).

NdM : du coup, ça nécessite d'être capable d'itérer efficacement sur les lignes accessibles depuis un stop donné à un temps donné.

NdM : ça nécessite également d'être capable de retrouver efficacement les différents trips d'une ligne en fonction de l'heure de passage.

**STEP 1** = à ce stade, on a le premier trip de chaque ligne passant par `p_src` (au sens large : ça inclut les stops `q` joignables à pied), i.e. on sait quels trips on peut attraper depuis le stop `p_src` en partant à `τ`.

> For each of those trips, if i < R(t), we add the trip segment pi → pR(t) to queue Q₀

VRAC : R(t) semble donc servir à retenir le premier stop permettant d'attraper un trip.

VRAC : un trip segment est un "morceau" de trip = une suite de stops d'un trip donné (et pour rappel : comme on parle de TRIP, on parle bien d'un TC à un datetime fixé !)

VRAC : Les différents `Qn` stockent les trips segments accessibles après `n` transferts. Du coup, ici, on ajoute chaque trip-segment à `Q₀` (vu qu'on n'a pas encore fait de transfert).

> and then update R(u) ← min(R(u), i) where t ≤ u  ∧  Lt = Lu

VRAC : C'est quoi déjà la relation de domination pour les trips ? En gros, `t ≤ u` veut dire que `t` et `u` sont deux trips d'une même ligne, et que `t` part avant.

En gros, pour chaque trip empruntable depuis `p_src`, on met également à jour les index des trips de la même ligne, qui partent après. C'est confirmé textuellement juste derrière :

> meaning we update the first reached stop for t and all later trips of the same line. [...] None of these later trips u can improve upon t. By marking them as reached, we eliminate them from the search and avoid redundant work.

Le point important, c'est que ces trips futurs "ne nous intéressent pas" (car ils sont dominés par `t`, donc moins intéressants que lui pour notre problème), donc à partir de maintenant, on les ignore.

**STEP 2** = tout ceci correspondait à la phase d'initialisation de l'algo. Le reste de l'algo consiste à dépiler les queues `Qx`, et traiter les trip-segments, jusqu'à ce qu'il n'en reste plus aucun.

### Dépilage des queues

Pour rappel, on a une queue par nombre de transferts :

- `Q₀` est la queue des trips-segments accessibles depuis le stop de départ `p_src`
- `Q₁` est la queue des trips-segments accessibles après un transfert
- etc.

Pour rappel, un trip-segment est un "morceau de trip", représenté par un stop de départ et un stop de fin dans un trip donné.

étape 1 = on regarde si le trip-segment atteint `p_tgt` ; pour cela, on utilise le set `L` des lignes capables d'atteindre le stop `p_tgt`. Si oui et s'il n'est pas déjà dominé, on ajoute au Pareto-set le trip, son heure d'arrivée, et son nombre de transferts.

étape 2 = on regarde si le trip peut-être pruned = si on a déjà trouvé un autre trajet permettant d'arriver plus tôt en `b+1` (i.e. il est inutile d'emprunter le trip, puisque dès le stop suivant, on connaît déjà un trajet plus efficace permettant de le rejoindre).

étape 3 (principale) = on regarde les transferts du trip-segment (NdM : attention, il y en a moins que les transfert du trip complet, puisqu'il faut se limiter aux transferts partant d'un stop `p ∈ [b, e]`). Pour chaque transfert, si ça nous permet d'améliorer l'index `R(u)` du trip à l'autre bout du transfert (i.e. on peut attraper le transfert plus tôt), alors on ajoute le transfert à `Qn+1`, et on met à jour l'index `R(v)` de tous les trips de la même ligne que `u` mais qui partent après (même principe que précédemment : l'idée est de ne pas les traiter, car ils ne sont pas intéressants).

QUESTION : à quoi correspond "avec les mains" l'index `R(t)` ? Stocke le stop le plus tôt où on peut prendre le trip (note : ou bien si un trip earlier a été atteint). En gros, ça permet de choisir de ne pas traiter un trip.

----

Du coup, le principe est de traiter tous les trips après, 0, 1, ... transferts. L'article fait le parallèle avec un BFS (où les trips sont les nodes du graphe, et les transferts sont les arêtes).

Comme on traite les nombres de transferts croissants, à chaque étape, on traite des trips avec PLUS de transferts que ceux traités jusqu'ici -> si à une étape N on ajoute un trip au Pareto-set (donc un trip non-dominé), c'est donc qu'il a permis d'arriver plus tôt que les étapes précédentes utilisant moins de transferts. Dit autrement : on trouve le trajet permettant d'arriver le plus tôt à la destination *en dernier*. Ceux avec moins de transferts qui ont été trouvés avant sont forcément plus longs (sans quoi on n'aura pas poursuivi le traitement, vu que tous les trajets futurs auraient été dominés en terme de nombre de transferts).

NdM : si je résume avec mes mots, ça ressemble beaucoup à RAPTOR :

- préliminaire = le preprocessing permet de connaître les transferts utiles entre les trips
- sur le principe, on itère sur les trips (en se limitant au premier trip de chaque ligne)
- on regarde où on peut aller avec 0 transferts (i.e. depuis le stop de départ `p_src`) avec les trips accessibles depuis le stop de départ
- puis, en utilisant les transferts preprocessés, on regarde où on peut aller avec les trips accessibles depuis `p_src + 1 transfert`
- puis, on regarde où on peut aller avec les trips accessibles depuis `(p_src + 1 transfert) + 1 transfert`
- etc.
- ce qui permet de savoir si un trip est intéressant ou non est l'index `R(t)` : si on a pu le rejoindre plus tôt, c'est qu'il a déjà été traité


NOTE : pour reconstruire le chemin final, on peut faire comme pour dijkstra, et mémoriser pour chaque trip-segment le "trip-parent" permettant de le rejoindre. Derrière, on reconstruit le chemin complet à l'envers, en partant du trip-segment permettant de rejoindre l'arrivée.
