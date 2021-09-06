# (ARTICLE) Round-Based Public Transit Routing

- **url** = [PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2012/01/raptor_alenex.pdf) (md5sum=`42f5bb72c05af994035bd3af8b3e75e5  `), [copie locale](./LOCALCOPIES/raptor_alenex.pdf)
- **source** = conf: [ALENEX 2012](https://archive.siam.org/meetings/alenex12/), proceedings: [Analysis of Experimental Algorithms](https://locus.siam.org/doi/book/10.1137/1.9781611972924)
- **auteurs** = Daniel DELLING, Thomas PAJOR, Renato F. WERNECK
- **date de publication** = 2012
- **date de rédaction initiale de ces notes** = juin 2020

**TL;DR** :
* RAPTOR = algo de calcul d'itinéraire TC :
    - DATA = GTFS (lignes, horaires, transferts piétons)
    - INPUT = station de départ + heure de départ + station d'arrivée
    - OUTPUT = Pareto-set des meilleurs itinéraires, optimisant l'EAT et le nombre de correspondances
* Caractéristiques :
    - efficace donc rapide, même sur des données volumineuses, comme celles de l'Île-de-France (ordre de grandeur sur Londres ~10ms)
    - sans preprocesing, donc notamment adaptable aux perturbations temps-réel
    - limitation n°1 = départ/arrivée limités à une STATION
    - limitation n°2 = pas de trajet piéton en cours de trajet, en dehors des transferts contenus dans la donnée GTFS
    - les deux limitations sont corrigées par les papiers de 2019 sur l'unrestricted-walking
* Contexte et caractéristiques :
    - Jusqu'au début des années 2010, le calcul d'itis TC était mal adressé, par des techniques graph-based (maintenant dépréciées, sauf éventuellement pour du multimodal)
    - RAPTOR est le premier algo efficace, et il y a eu d'autres inventions depuis
    - [CSA](https://arxiv.org/pdf/1703.05997.pdf) inventé en 2013 = caractéristiques assez similaires à RAPTOR, possiblement un peu plus rapide, mais pas clair sur la capacité à renvoyer un pareto-set optimisant également le nombre de transferts
    - [Transfer Patterns](https://ad-publications.cs.uni-freiburg.de/ESA_transferpatterns_BCEGHRV_2010.pdf), inventé en 2010, utilisé par google -> le plus rapide (scale même à l'échelle nationale), mais nécessite un lourd préprocessing.
    - [Trip-Based Public Transit Routing](./2015-trip-based-public-transit-routing.md), inventé en 2015, plus rapide que RAPTOR, mais en n'ayant que très peu de preprocessing, donc intermédiaire entre RAPTOR/CSA et TransferPatterns.
    - diverses labeling techniques ([exemple1](https://dl.acm.org/doi/10.1145/2723372.2749456), [exemple2](https://link.springer.com/chapter/10.1007%2F978-3-319-20086-6_21), ...)
* Utilisé entre autre par [le moteur navitia](https://github.com/CanalTP/navitia).
* Rapide description avec les mains :
    - l'algo fonctionne en rounds, et maintient pour chaque stop un earliest-arrival time ; tous sont initialisés à `+∞`
    - initialisation de l'algo = on met à jour le stop de départ, en settant son EAT au departure-time
    - on commence chaque round avec un set de stops marqués, i.e. de stops dont l'EAT a été amélioré au round précédent
    - pour le round courant, on liste toutes les routes qui empruntent un stop précédemment marqué, et on met à jour les stops atteignables par ces routes
    - (on met également à jour les stops joignable via un transfert piéton depuis un stop atteignables par ces routes)
    - si la mise à jour d'un stop améliore son EAT, on le marque pour le round suivant
    - au final, à chaque round k, on a le meilleur EAT auquel on peut atteindre chaque stop, en n'utilisant que k trips
    - l'algo s'arrête lorsqu'aucun stop n'a été marqué par le dernier round (i.e. on ne peut plus améliorer l'EAT d'aucun stop)
* À partir d'un stop SOURCE (S) donnée, le résultat de l'algorithme, c'est les EAT en un nombre de transferts donnés, pour chaque stop possible, y compris le stop TARGET (T) :
    - `Round 1: S->T = 1400            <-- en un seul trip, on arrive à 1400`
    - `Round 2: S->A->T = 1300         <-- en deux trips, via A, on va plus vite qu'en un trip, et on arrive à 1300`
    - `Round 3:                        <-- en trois trips, on ne fait pas mieux qu'en deux trips`
    - `Round 4: S->A->B->C->T = 1200   <-- en quatre trips, via A+B+C, on va plus vite qu'en deux trips, et on arrive à 1200`
    - Résultat final = on renvoie un Pareto-set contenant TROIS itinéraires.
* le papier décrit deux extensions :
    - McRAPTOR = extension de RAPTOR pour le multi-critère (e.g. le prix)
    - rRAPTOR = extension de RAPTOR pour les range-queries (aussi appelées profile-queries dans d'autres papiers) = au lieu de prendre en input le datetime de départ, on prend en input un range de datetimes
* les annexes donnent des précisions sur comment implémenter efficacement l'algo
* pas directement relié au papier, mais voici [un post](https://ljn.io/posts/raptor-journey-planning-algorithm) vulgarisant RAPTOR, et donnant des corrections et alternatives (ce poste confirme l'erreur dans le papier sur la prise en compte des footpaths dans le pseudo-code)

[[_TOC_]]

## Section 1 : Introduction

* Les 2 critères de raptor = EAT + number of transfers
* Calcule tous les Pareto-optimal journeys
* Parallélisable !
* Dans le même papier : McRAPTOR = extension de RAPTOR pour prendre en compte plus de critères (e.g. prix)
* Dans le même papier : rRAPTOR = extension de RAPTOR pour range queries = au lieu de prendre en input le datetime de départ, on prend en input un range de datetimes
* Pas de preprocessing nécessaire = autorise les modifications dynamiques sur les GTFS en entrée

## Section 2 : Preliminaries

Signification des données :
* stop = une station de TC. Formellement, une coordonnée géographique ou on peut monter/descendre d'un TC
* trip = un subset de N stops chacun étant associé à une horaire de passage
    - c'est un "vehicle-trip" : il faut voir le trip comme "un trajet d'un véhicule physique à une horaire donnée"
    - exemple de trip = le RER EKFI qui emprunte la ligne B1 en partant de ROBINSON à 8h57 et arrivant à CDG à 10h12, omnibus
* route = un ensemble de stops appartenant à au moins un trip
    - chaque trip appartient à exactement une route
    - deux trips d'une même route empruntent EXACTEMENT les mêmes stations, mais à des horaires différentes
    - exemple de route = la ligne de RER B1 qui est omnibus de ROBINSON à CDG (le trip donné en exemple plus haut appartient à cette route)
    - exemple DIFFÉRENT de route = la ligne de RER B1 qui va aussi de ROBINSON à CDG, mais est direct entre Massy-Palaiseau et Bourg-la-Reine
* Ma compréhension = les différents trips d'une même route ont EXACTEMENT les mêmes séries de stops, mais à des horaires différents.
* Ma compréhension = une route est ordonnée : pour représenter une ligne RER, il faut deux routes : une dans chaque direction

timetable = le jeu de données en entrée, agrégat de 5 items :
* Π = est le set des datetimes possibles (périodique), e.g. les 86400 secondes dans un jour
* S = les stops  (= les stations)
* T = les trips  (= subset de stops, chacun étant associé à une horaire de passage)
* R = les routes (= regroupement de trips qui ont les mêmes stations à des horaires différentes)
* F = les footpaths (= les transferts à pied entre des stations proches)

au sein d'un trip, chaque stop se voit associé DEUX temps :
* τarr = heure d'arrivée au stop
* τdep = heure de départ du stop
* (on a τarr <= τdep, et entre les deux, les voyageurs montent ou descendent)
* le premier stop d'un trip a un τarr indéfini (et le dernier stop d'un trip a un τdep indéfini)

l'ensemble des routes est une partition de l'ensemble des trips (chaque trip appartient à une seule route, et les routes recouvrent tous les trips)

chaque footpath a deux stops p1 et p2, et le temps de transfert est une constante l(p1, p2) 

F est transitively-closed : si p1 est relié à p2, et p2 est relié à p3, on a toujours p1 relié à p3

output de l'algo :
* sortie de l'algo = un set de journeys
* un journey = une séquence de trips + footpaths. Les trips sont assortis d'un pick-up stop et d'un drop-off stop.
* note importante pour l'algo : pour un journey qui contient k trips, on a exactement k-1 transferts (éventuellement instantanés)
* Pareto-set = set de journeys qui ne se dominent pas les uns les autres

input de l'algo pour earliest-arrival problem :
* source stop ps
* target stop pt
* departure time τ
* mono-criteria = on recherche un journey qui part de ps à τ ou après, et qui arrive à pt le plus tôt possible
* multi-criteria = à cet earliest-arrival time, on ajoute d'autres critères d'optimisation (e.g. minimiser le nombre de transferts) -> on recherche un Pareto-set

input de l'algo pour range problem :
* source stop ps
* target stop pt
* intervalle Δ ∈ Π
* revient à résoudre un earliest-arrival problem pour chaque τ ∈ Δ

2.1 = existing graph-based approach
* time-expanded approach = on consruit un énorme graphe avec tous les events possibles
* time-dependent approach = les trips d'une même route forment un seul edge, time-dependent, conduit à un graphe plus efficace
* dijkstra modifié pour respecter le multi-critère (au lieu d'associer un coût à un noeud, on associe un pareto-set)
* autre alternative = layered dijkstra, qui fonctionne quand le second critère d'optimisation est discret (comme le nb de transferts)
* CE QU'IL FAUT EN RETENIR : c'est qu'il existe des alternatives (a priori, moins efficaces que RAPTOR)

## Section 3 : RAPTOR

Recall that our goal is to compute for every k a nondominated journey to a target stop p t with minimum arrival time having at most k trips.
* ça répond (pour le bi-critère, au moins) à ma question de "comment RAPTOR limite le nombre de trajets renvoyés"
* en effet, il renvoie TOUS les trajets non-dominés, pour tous les k possibles
* (le nombre de trajets est "mécaniquement" limité : un trajet qui a 19 transferts sera probablement dominé par un trajet qui n'en a que 2)

L'algo fonctionne par rounds, au k-ième round, on sait aller partout avec un temps de parcours optimal en prenant k trips (donc avec k-1 transferts).

> More precisely, the algorithm associates with each stop p a multilabel (τ0(p), τ1(p), … , τK(p)) where τi(p) represents the earliest known arrival time at p with up to i trips. All values in all labels are initialized to +∞.

NdM : si τ1(p) reste à +∞ à la fin de l'algo, c'est qu'il est impossible d'atteindre p depuis la source en 1 trip, i.e. sans faire de transfert

on initialise τ0(ps) = τ   (τ0 = avec 0 trip -> tous les τ0(p) seront à +∞, sauf τ0(ps) qui sera égal à τ)

invariant = au début du round k, les τ0…τk-1 sont corrects. Dit autrement : pour chaque stop p, τi(p) est réellement l'earliest-arrival à p en utilisant i-1 transferts

objectif du round k = pour tous les p, calculer les τk(p) définitifs
    stage1: τk(p) <- τk-1(p)
    stage2: on itère sur toutes les routes du timetable
    - soit r une route
    - soit T(r) = t0, t1, ... tN  la liste de ses trips, classés par datetime de parcours
    - on s'intéresse aux journeys où le k-ième (donc le dernier) trip appartient à la route r
    - et(r, pi) est le trip t le plus tôt de la route r qu'on peut attraper au stop pi, l'earliest  τdep(t, pi) >= τk-1(pi)
    - le t correspondant est le "current trip" pour la route pour k
    - pour process une route, on itère dans l'ordre sur ses stops, jusqu'à ce qu'on en trouve un ou et(r, pi) est défini
    - et(r,pi) est là où on peut "monter" dans la route. On mémorise (pour pouvoir reconstruire le journey complet) le stop auquel on est monté dans t
    - QUESTION : pas encore super clair la représentation de tout ça
        + je pense que pour un round k, et(r,pi) est l'endroit où on peut entrer dans la route au plus tôt depuis le stop source en k-1 transferts
        + gardons à l'esprit, que tout ça, c'est pour UN ROUND k (ces calculs de et(r,pi) sont faits pour un k donné, et refaits à chaque round)
    - on peut alors mettre à jour tous les τk(pi) pour tous les stops pi d'une route donnée qui sont après celui de et(r,pi)
    - le long d'une route, il peut y avoir un et(r,pi) pour un p donné, et un autre et(r,pi) pour un autre p donné ; dit autrement : le long d'une route, on peut arriver à certains stops plus tôt que d'autres)
    stage3: on met à jour pour prendre en compte les footpaths :
    - on itère sur les footpaths (pi, pj) ∈ F
    - on sette τk(pj) = min{τk(pj), τk(pi) + l(pi,pj)}

condition d'arrêt de l'algorithme = si lors d'un round k, aucun τk(p) n'a été amélioré, on peut s'arrêter

complexité : chaque round est linéaire en l'addition de :
* la somme des stops dans toutes les routes
* le nombre de trips
* le nombre de footpaths

le papier suggère en annexe des moyens d'implémentation efficace (accès en temps constant dans des arrays)

3.1 = optimisations :
* ne pas explorer les routes si ça ne sert à rien
* ne pas explorer les stops d'une route s'ils sont inutiles
* local pruning = s'arrêter quand on sera sûr d'être dominés par les itérations futures
* target pruning = on peut ignorer tous les stops qui nous feront arriver après notre meilleur ETA à la station target

3.2 = transfer preferences = à deux trajets égaux (en nombre de transferts + en temps de trajet), on peut préférer un tranfert plutôt qu'un autre
3.3 = parallelization = au sein d'un round, chaque route peut être traitée en parallèle ; proposition sympa utilisant un graphe (node=route / edge entre deux nodes s'ils ont des stops communs) et sa coloration pour créer des buckets de route traitables en parallèle

## Section 4 : Extensions

Je n'ai fait que survoler :
* mcRAPTOR = multi-criteria RAPTOR
* rRAPTOR = range-RAPTOR

## Section 5 : Experiments

* réseau de Londres :
    - ~20k stops
    - ~2k routes
    - ~133k trips
    - ~5M distinct departures (a trip departing at a stop)
    - ~45k footpaths
* évaluation = 10000 queries sur des paires source+target au hasard
* en moyenne, 8,4 round avant d'arrêter, chaque route est explorée 3 fois en moyenne
* average query time = 7.3 ms
* Le papier donne des références vers des données GTFS d'autres villes.

## Notes sur le pseudo-code


*ATTENTION* : il y a un bug dans le papier original de RAPTOR sur le pseudo-code de prise en compte des transferts à pieds : il ne faut pas prendre les footpaths du round k, mais du round k-1.  cf. aussi [ce blogpost](https://ljn.io/posts/raptor-journey-planning-algorithm) qui en parle.

PHASE 1 = convertir les stops marqués en doublet (r,p)
* à chaque round, on traite les stops marqués par le round précédent
    - les stops marqués p sont ceux qui ont vu leur τi(p) améliorés au round précédent
    - au premier round de l'algo, seul le stop source ps est marqué
* pour chaque stop marqué, on ajoute un doublet (r, p) dans une queue Q avec toutes les routes desservant le stop marqué
    - subtilité : si une même route r dessert PLUSIEURS stops marqués on n'ajoute qu'UN SEUL doublet, celui ayant le stop le PLUS TÔT sur la route
* à ce stade, on a toutes les routes desservant les stops marqués, en ignorant les stops marqués d'une route qui ne sont pas les plus tôt

PHASE 2 = mettre à jour les stops pi qui suivent p sur la route r, pour tout doublet (p,r) de la queue
* on itère sur tous les doublets (p,r) enqueués à la PHASE 1
* pour chacun, on itère sur tous les stops pi de r, QUI ARRIVENT APRÈS p
* PAS CLAIR : quel est le trip dénommé ⊥ dans le papier ?
* PAS CLAIR : c'est quoi τ*(p) dans le papier ?
    - the best known arrival time at p
    - ça sert pour le local pruning : rien ne sert de marquer les stops au round k si on n'améliore pas leur meilleur temps trouvé jusqu'ici
    - en effet, on aurait au round k un temps moins bon qu'à un round k-1, et donc tout chemin utilisant ce stop serait dominé (temps de parcour + nb transfert)
    - ça sert également pour le target pruning : rien ne sert de marquer les stops si on y arrive déjà après le meilleur temps vers le stop target pt
    - du coup, cette formule matérialise à la fois le local-pruning et le target-pruning  :  arr(t, pi) < min{τ∗(pi),τ∗(pt)}
* pour chaque stop pi sur lequel on itère, on améliore τk(pi) et τ*(pi) si on peut, et dans ce cas, on marque pi pour le round suivant
* PAS CLAIR : c'est quoi et(r,pi) ?
    - Let et(r,pi) be the earliest trip in route r that one can catch at stop pi , i.e. the earliest trip t such that τdep(t,pi) ≥ τk−1(pi)
* in fine, même si le pseudo-code est pas clair, je suppose qu'on retient le trip t sur la route r qui permet de partir au plus tôt de pi

PHASE 3 = traitement des footpahts
* pour chacun des stops pi qui ont été améliorés (donc marqués) par le round actuel, on traite leurs footpaths
* si parmi les stops pi améliorés, on peut rejoindre un stop pj à pied et si ce faisant on améliore son τ*(pj) alors on met à jour pj (et on le marque)

PHASE 4 = s'il n'y a pas eu de stops améliorés, l'algo est fini
