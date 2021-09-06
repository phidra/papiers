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

Ici, on dirait qu'il s'agit des lignes capables de rejoindre le stop `p`... lequel ? Le stop cible `p_tgt` ? Ou bien n'importe quel stop ? L'article parle bien de `p_tgt`.

D'après la formule qui suit, on dirait qu'on retient :

- toutes les lignes passant par le stop `p_tgt`
- ainsi que toutes les lignes passant par un autre stop `q` pour lequel il existe un footpath permettant de rejoidnre `p_tgt` depuis `q`.

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
