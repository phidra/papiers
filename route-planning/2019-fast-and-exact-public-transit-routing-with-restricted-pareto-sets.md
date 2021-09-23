# (ARTICLE) Fast and Exact Public Transit Routing with Restricted Pareto Sets

- **url** = [lien](https://epubs.siam.org/doi/abs/10.1137/1.9781611975499.5), [PDF](https://epubs.siam.org/doi/pdf/10.1137/1.9781611975499.5) (md5sum=`ed3a94cf0dfd4da392760f1b17e3dec6`), [copie locale](./LOCALCOPIES/1.9781611975499.5.pdf)
- **source** = conf: [ALENEX 2019](https://www.siam.org/conferences/cm/conference/alenex19), [proceedings](https://epubs.siam.org/doi/book/10.1137/1.9781611975499)
- **auteurs** = [Daniel DELLING](https://www.researchgate.net/scientific-contributions/Daniel-Delling-20515954), [Thomas PAJOR](https://www.researchgate.net/scientific-contributions/20516545-Thomas-Pajor), [Julian DIBBELT](https://www.researchgate.net/profile/Julian_Dibbelt), ce sont des noms connus (DIBBELT est un ancien du KIT, notamment), il semblerait qu'ils travaillent dorénavant chez Apple.
- **date de publication** = 2019
- **date de rédaction initiale de ces notes** = septembre 2021

**TL;DR** :

* le TL;DR reste à faire

* [(ARTICLE) Fast and Exact Public Transit Routing with Restricted Pareto Sets](#article-fast-and-exact-public-transit-routing-with-restricted-pareto-sets)
* [Abstract](#abstract)
* [section 1 = Introduction](#section-1--introduction)
* [section 2 = Preliminaries](#section-2--preliminaries)
* [Section 3 = restricted pareto set](#section-3--restricted-pareto-set)
* [Section 4 = Bounded McRAPTOR](#section-4--bounded-mcraptor)
   * [Approche n°1 = self-BMRAP](#approche-n1--self-bmrap)
   * [Approche n°2 =](#approche-n2-)

# Abstract

- le problème adressé est le problème MULTI-CRITÈRES ; un RAPTOR classique est déjà bi-critère (EAT + nombre de correspondances), mais là on parle d'encore plus de critère, comme p.ex. le temps de marche à pied
- le pareto-set (= front de Pareto) peut être très grand, vu que le troisième critère peut être continu (e.g. marche à pied)
- l'objectif du papier semble être de réduire le pareto-set :
    1. ils formalisent la définition d'un restricted pareto-set (de façon indépendante de l'algo utilisé pour le calculer)
    2. ils proposent un algo efficace, dérivé de McRAPTOR, pour calculer ce restricted pareto-set (en vrai, une famille d'algos = Bounded-McRAPTOR)
- résultats : pour calculer un pareto-set à 4 critères, leur algo va 65 fois plus vite qu'en calculant le pareto-set non-restreint, tout en conservant les trajets importants

# section 1 = Introduction

- calculer un front de Pareto bi-critère est NP-hard en général (mais faisable pour les réseaux de transport publics)
- les techniques metionnées sont :
    - CSA
    - RAPTOR
    - TransferPatterns
    - TripBasedPublicTransitRouting
    - Graph-labeling (je ne connais pas bien cette technique, correspondant à Public-Transit-Labeling)
- ces techniques sont efficaces à 2 critères, mais beaucoup moins efficaces à plus de critères, pour deux raisons :
    - 1. l'algo doit maintenir des structures de données plus compliquées
    - 2. mécaniquement, l'ajout de critères augmente la taille du Pareto-set, possiblement fortement
- quelques critères évoqués, pour lesquels il peut être utile de calculer un front de Pareto :
    - temps de marche à pied
    - fiabilité des TC (certains TC seront moins fiables ?)
    - coûts
    - accessibilité (e.g. privilégier les stations accessibles aux PMR ?)
    - nombre de bus
- possibilité = ignorer des résultats légèrement inférieurs à d'autres sur le front de Pareto
    - ça marche bien si on l'applique À LA FIN de l'algo, pour raffiner un front de Pareto qui a été calculé exhaustivement
    - mais si on l'applique en cours d'algo, on peut très bien louper des algos importants in fine
    - et comme l'objectif est d'éviter de calculer un front de Pareto exhaustif, cette approche n'est pas très utile
- idée principale du papier :
    - calculer le front de Pareto bi-critère complet et exact
    - n'autoriser dans le front de Pareto multi-critère réduit QUE des itis qui ne dégradent pas trop l'EAT ou le nombre de transfers
    - Bounded McRAPTOR utilise RAPTOR et McRAPTOR comme briques de base
- tight-BMRAP = l'une des variantes de la famille Bounded-McRAPTOR:
    - fait une série de RAPTOR normaux pour précalculer des limites (bounds)
    - puis fait un McRAPTOR normal qui utilise les bounds pour pruner des trajets 
- ils annoncent un résultat intéressant = il "suffit" de doubler le temps de calcul pour calculer un RAPTOR qui optimise les 4 critères suivants :
    - EAT
    - nombre de transferts
    - temps de marche à pied
    - nombre de bus
- note : tout comme RAPTOR, pas besoin de preprocessing

# section 2 = Preliminaries

- rappel des notations
- rappel de RAPTOR :
    - on fonctionne en rounds
    - à chaque round, on regarde toutes les routes qui passent par les stops améliorés au round précédent, et plus précisément le premier trip de la route qu'on peut attraper à ce stop
    - si on peut y arriver plus tôt grâce à ce trip, on met à jour les stops de chacun de ces trips
    - avant de passer au round suivant, on relaxe les transferts piétons de chaque stop amélioré
    - note : en faisant tourner l'algo à l'envers, on peut calculer des trajets qui "arrivent à telle heure", plutôt que des trajets qui "partent à telle heure"
    - possiblement en lien avec l'unrestricted-walking : il y a un paragraphe sur le fait que le graphe piéton ne soit pas constitué de cliques (ils contournent le problème en maintenant un set de labels piétons)
- rappel de McRAPTOR :
    - là où RAPTOR maintient pour chaque stop un label indiquant le meilleur temps d'arrivée en k legs TC, McRAPTOR maintient pour chaque couple `{stop,round}` un **BAG** de labels, où tous les labels du bag sont sur le front de Pareto (i.e. aucun n'est dominé)
    - chaque label est un tuple contenant N éléments (où N est le nombre de critères à optimiser, sauf le nombre de correpsondances, déjà pris en compte par la notion de round)
    - la façon dont on scanne les routes semble également modifiée, pour prendre en compte les bag de labels
    - (on dirait qu'on généralise le fait de trouver le meilleur trip, en pouvant trouver LES meilleurs trips, un pour chaque label dans le bag)
    - cause de lenteur n°1 = McRAPTOR est 4 fois plus lent que RAPTOR de base (i.e. même sans ajouter de critère supplémentaire) car on a des structures plus complexes à gérer
    - cause de lenteur n°2 = le merge des bags de labels (pour éliminer ceux qui sont dominés) est quadratique en le nombre de labels (qui sont d'autant plus nombreux qu'on a de critères)
    - du coup, McRAPTOR peut vite devenir trop lent pour être utilisé en pratique

# Section 3 = restricted pareto set

- le front de Pareto "exhaustif" contient TOUS les trajets non-dominés
- problème : il suffit d'améliorer le temps de marche à pied de 30 secondes pour inclure dans le front de Pareto un trajet qui dégrade l'EAT de plusieurs heures... Or, c'est un trajet que l'utilisateur n'empruntera jamais en pratique, car rallonger son trajet de plusieurs heures pour gagner 30 secondes de marche à pied n'est jamais intéressant pour un utilisateur...
- ils donnent un exemple illustratif, où on multiplie par trois le temps de trajet total (avec un trajet alambiqué, du coup), pour ne marcher que 250 mètres au lieu de 500 mètres...
- typiquement : une fois le pareto-set obtenu, les itis du front de Pareto sont classés, et on ne les présente pas tous à l'utilisateur
- (NdM = une autre façon de voir les choses, pertinente dans le cadre de cet article : en calculant le Pareto-set exhaustif, on a calculé beaucoup d'itis inintéressants)
- un point intéressant = l'approche intuitive consistant à essayer de pruner les labels en cours d'exécution de l'algo ne marche pas
    - on aurait pu essayer de dégager un label L2 qui dégrade fortement un critère et n'améliore que marginalement le critère qui le place sur le front de Pareto
    - ça ne marche pas 1 = on n'a pas de garantie sur les labels qui "survivront" en fin d'exécution
    - ça ne marche pas 2 = c'est très dépendant de détails d'implémentation de l'algorithme (deux algos différents renverront un front de Pareto restreint différent)
- à l'inverse, l'approche de l'article garantit que deux algos différents renverront un front de Pareto restreint identique
- objectif = dégager les itis qui offre un compromis inintéressant (et auraient été dégagés lors de la phase de post-processing du front de Pareto, donc jamais montrés à l'utilisateur final)
    - on commence par calculer le front de Pareto "anchor", i.e. le front de Pareto classique à deux critères EAT+transfers (les itis sur ce front de Pareto sont les `anchor-journeys`)
    - notation : `J*` est un **anchor-journey**, `τarr(J*)` est son EAT, et `tr(J*)` est son nombre de trips
    - principe = chaque itinéraire sur le restricted-pareto-set ne doit pas dégrader significativement `τarr(J*)` et `tr(J*)`
    - étant donné un iti `J` (sur le front de Pareto exhaustif avec plus que deux critères), son `corresponding anchor-journey` est :
         - parmi les itis `J*` qui ont autant de trips ou moins de trips que `J`, le `corresponding` est celui qui a le PLUS de trips
         - ça peut être le même iti (`J = J*`), dans ce cas, `J` est également sur le front de Pareto `anchor`
         - ça peut être des itis différents, p.ex. `J*` a autant de correspondance, et arrive plus tôt que `J` (car il a ignoré le critère supplémentaire qui a fait que `J` est sur le front de Pareto exhaustif)
    - À CLARIFIER : mieux comprendre pourquoi `J*` est toujours unique, et comment le trouver...
- derrière, en gros, le `restricted pareto-set`, c'est un intermédiaire entre `J*` et `J`, dans lequel tout iti restreint est à au plus `σarr` et `σtr` de son `corresponding-anchor-journey`)
    - (dit autrement, `J` ne "dégrade" son `corresponding anchor-journey` que de `σarr` et/ou `σtr`)
    - définir `σarr` (resp. `σtr`) entre `0` et `∞` permet de positionner le front de Pareto restreint entre le front de Pareto anchor et le front de Pareto exhaustif
- représentation graphique intéressante = quand on graphe le nb de transfers vs. l'EAT, le front de Pareto restreint est défini par des rectangles autour du front de Pareto anchor

# Section 4 = Bounded McRAPTOR

Il y a plusieurs types d'algos dans la famille, selon qu'ils permettent `σtr ≠ +∞` (i.e. qu'ils respectent le caractère restreint du Pareto-set sur le critère du nombre de transfers), et selon qu'ils calculent le restricted Pareto set exact (ou un superset plus large, contenant le restricted pareto-set).

## Approche n°1 = self-BMRAP

En gros, c'est un McRAPTOR classique, sauf que :

- on maintient le meilleur EAT au stop cible `τ*` (on commence avec `τ* = +∞`, et on le mets à jour à chaque round)
- en cours d'algo, on peut pruner les trajets lorsque leur arrivée à un stop intermédiaire est déjà supérieur à `τ* + σarr`

Problème 1 = pour pruner efficacement, ça n'est pas le `τ* `en cours d'algo qu'il faudrait utiliser, mais le meilleur EAT final (or, on ne le connaît pas encore). En effet, si le meilleur EAT n'est calculé qu'au dernier round, on aura passé tous les rounds suivant à ne pruner qu'avec un EAT trop tardif, et donc à garder dans le front de Pareto des itis qui auraient été à dégager avec l'EAT final.

Problème 2 = ça ignore complètement `σtr`, et ne se concentre que sur `σarr` -> on ne calcule pas vraiment le restricted-pareto-set, mais plutôt un surensemble.

## Approche n°2 = target-BMRAP

En gros, c'est la même approche que self-BMRAP, mais qui corrige son problème n°1, en lançant d'abord un RAPTOR classique pour connaître le meilleur EAT (ou plutôt, le set de meilleurs EAT en fonction du nombre de transfers, i.e. le front de Pareto classique), puis en utiliant ce(s) meilleur(s) EAT pour pruner les trajets au cours de l'exécution du McRAPTOR.

Le pruning devient exact pour le rapport à `σarr`, mais on n'est toujours pas capable de prendre en compte `σtr`.

## Approche n°3 = tight-BMRAP

TO BE CONTINUED
