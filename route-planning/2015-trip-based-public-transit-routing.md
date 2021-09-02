# (ARTICLE) Trip-Based Public Transit Routing

- **url** = [lien](https://arxiv.org/abs/1504.07149), [PDF](https://arxiv.org/pdf/1504.07149.pdf) (md5sum=`34f635be34a406929c725f912ddaef0e`), [copie locale](./LOCALCOPIES/1504.07149.pdf)
- **source** = conf: [ALGO/ESA 2015](http://algo2015.upatras.gr/esa/accepted.html), [proceedings](https://link.springer.com/book/10.1007%2F978-3-662-48350-3)
- **auteurs** = [Sascha WITT](https://algo2.iti.kit.edu/english/2373.php) (Karlsruhe Institute of Technology)
- **date de publication** = 2015
- **date de rédaction initiale de ces notes** = septembre 2021

**TL;DR** :

* le TL;DR reste à faire

# Transfert de mes notes brutes

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
