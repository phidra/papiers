# (ARTICLE) Integrating ULTRA and Trip-Based Routing

- **url** = [lien](https://drops.dagstuhl.de/opus/volltexte/2020/13140/), [PDF](https://drops.dagstuhl.de/opus/volltexte/2020/13140/pdf/OASIcs-ATMOS-2020-4.pdf) (md5sum=`5d3f94e9919427af8e848ff8f8dbeb81`), [copie locale](./LOCALCOPIES/OASIcs-ATMOS-2020-4.pdf), [code github](https://github.com/kit-algo/ULTRA-Trip-Based)
- **source** = conf: [ATMOS 2020](http://algo2020.di.unipi.it/ATMOS2020/index.html#home), [proceedings](https://drops.dagstuhl.de/opus/portals/oasics/index.php?semnr=16169)
- **auteurs** = [Jonas SAUER](https://i11www.iti.kit.edu/members/jonas_sauer/index), [Dorothea WAGNER](https://i11www.iti.kit.edu/en/members/dorothea_wagner/index), [Tobias ZÜNDORF](https://i11www.iti.kit.edu/members/tobias_zuendorf/index), tous trois du KIT (Karlsruhe Institute of Technology)
- **date de publication** = 2020
- **date de rédaction initiale de ces notes** = septembre 2021

**TL;DR** :

* le TL;DR reste à faire

# Vidéo explicative

Sur [la chaîne YouTube d'ALGO 2020](https://www.youtube.com/channel/UCBvRy0gXDEQaf_dl8UUAE7g), il y a une vidéo dédiée à ULTRA-TB : [lien](https://www.youtube.com/watch?v=40QXUYfLuQ4).

Notes en vrac à la consultation de la vidéo :

* combinaison de deux graphes :
    * graph n°1 = timetabled based = PTN (en gros, le TC)
    * graph n°2 = unscheduled = transfer graph (en gros, le "walk" au sens large)
* ULTRA = précalculer des shortcuts qui remplacent le graph n°2 par un set de shortcuts (+ bucketCH au départ/arrivée)
* Trip-Based routing = précalculer des shortcuts entre les trips (c'est une vision intéressante)
* différence entre les shortcuts ULTRA et les shortcuts TB :
    * shortcuts ULTRA = time-independent
    * shortcuts TB = time-dependent
* dans les deux cas, le preprocessing consiste à énumérer les journeys qui ont entre 0 et 2 trips inclus
* Pour s'intéresser à un transfert donné (transfert de la ligne bleue à la ligne verte, entre s' et t :
    * ULTRA énumère tous les journeys ayant entre 0 (= full-walk) et 2 trips inclus, permettant de faire le même trajet de s' à t'
    * le shortcut entre la ligne bleue et la ligne verte n'est gardé que si le journey l'utilisant est le meilleur parmi tous ceux énumérés
    * ULTRA compare les journeys avec le même FIRST-STOP
    * Trip-Based compare les journeys avec le même FIRST-TRIP (ce qui impose donc la ligne bleue)
    * du coup, les journeys énumérés par ULTRA pour savoir s'il faut ou non garder un shortcut ENGLOBENT les shortcuts énumérés par Trip-Based
* Loup : la définition de "better alternative" (pour savoir s'il faut garder le shortcut ou non) est tricky, ils réfèrent au papier.
* Trip-based query (étendue à ULTRA) :
    * après une bucket-CH pour savoir à quelle heure on peut arriver à chaque stop,
    * on regarde pour chaque stop quel est le premier prochain trip qu'on peut prendre
    * pour Trip-Based "classique" c'est rapide car on n'a à faire ça qu'avec un seul stop (le stop source)
    * pour ULTRA-Trip-Based, c'est plus long, car il faut le faire pour tous les stops (vu qu'ils sont tous théoriquement atteignables à pied, même si pratiquement, ça peut être démesurément long d'y aller à pied)
    * pour accélérer cette recherche de "quel est le premier trip que je peux attrapper à chaque stop" (NdM : pas encore super clair...) :
        * on regarde pour le stop source quel est le premier prochain trip qu'on peut prendre
        * on utilise les routes de RAPTOR pour regarder tous les autres stops de la route qu'on peut atteindre à partir de ce premier prochain trip du stop source
        * pour chaque autre stop, on regarde quels sont les trips qu'on peut atteindre (qui peuvent être plus tôt, selon l'heure à laquelle on arrive en marchant)
    * en gros, on utilise la requête trip-based mais en l'initialisant avec tous les trips qu'on peut utiliser à tous les stops (plutôt que simplement les trips du stops source)
* Le fait de combiner le preprocess ULTRA et Trip-Based est un peu plus long (17%), MAIS génère beaucoup moins de shortcuts (3 à 9 fois moins).
* ULTRA-TB est plus rapide qu'ULTRA-RAPTOR (et également qu'un Trip-Based sans unrestricted-walking !)
