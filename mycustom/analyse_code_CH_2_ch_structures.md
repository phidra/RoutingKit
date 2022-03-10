* [Les structures de CH](#les-structures-de-ch)
   * [Préambule = identification des nodes dans les graphes ch.forward et ch.backward](#préambule--identification-des-nodes-dans-les-graphes-chforward-et-chbackward)
   * [Les structures de ch.forward / ch.backward](#les-structures-de-chforward--chbackward)
      * [Bug probable](#bug-probable)
   * [Construction de shortcut_first_arc / shortcut_second_arc](#construction-de-shortcut_first_arc--shortcut_second_arc)
   * [Comment unpacker un shortcut ?](#comment-unpacker-un-shortcut-)
   * [Comment unpacker un shortcut — compléments sur shortcut_second_arc](#comment-unpacker-un-shortcut--compléments-sur-shortcut_second_arc)

# Les structures de CH

**NOTE** : cette page s'intéresse à ce que représentent les structures de la classe `ContractionHierarchy` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/contraction_hierarchy.h#L53)) :

```cpp
struct Side{
    std::vector<unsigned>first_out;
    std::vector<unsigned>head;
    std::vector<unsigned>weight;

    BitVector is_shortcut_an_original_arc;
    std::vector<unsigned>shortcut_first_arc; // contains input arc ID if not shortcut
    std::vector<unsigned>shortcut_second_arc;// contains input tail node ID if not shortcut
};

std::vector<unsigned>rank, order;
Side forward, backward;
```

## Préambule = identification des nodes dans les graphes ch.forward et ch.backward

**Point important** : dans les graphes `ch.forward` et `ch.backward`, les nodes sont identifiés par leur *rank* (plutôt que par leur *order*).

C'est assez logique, vu que c'est ça qui nous intéresse à la base pour savoir si on a le droit de les parcourir ; mais comme ça n'est PAS comme ça que `ch.forward` (resp. `ch.backward`) sont construits (en effet, ils sont construits avec les IDs de node, et non leur rank ! ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L689))), ça m'a confusé au début.

Il est probable qu'une fonction qui vient après `build_ch_and_order` post-processe la structure `ch` pour replacer les ids de nodes par leurs ranks. C'est sans doute l'une de ces 3 fonctions :

- `optimize_order_for_cache` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L902)
- `make_internal_nodes_and_rank_coincide` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L833)
- `sort_ch_arcs_and_build_first_out_arrays` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L864)

## Les structures de ch.forward / ch.backward

J'ai pu vérifier que dans chaque graphe (`ch.forward` ou `ch.backward`) de la CH, on stocke des infos **sur les edges** :

- `first_out` contient N+1 éléments (où N = nombre de nodes), et référence les **edges** d'un node (cf. structure `AdjacencyArray`, pour itérer sur les out-edges d'un node, il faut itérer entre `first_out[node]` et `first_out[node+1]`)
- derrière, **TOUTES** les autres structures contiennent E éléments (où E = nombre d'edges), un par edge dans les graphes CONTRACTÉS `ch.forward` ou `ch.backward` :
    * `head` est le noeud de destination de l'edge
    * `weight` est son poids
    * `is_shortcut_an_original_arc` (que j'aurais plutôt appelé _is_edge_an_original_arc_)  est un booléen indiquant si l'edge dans `ch.forward` est réel (présent dans le graphe original) ou shortcut (généré par la contraction d'un node)
    * `shortcut_first_arc` (resp. `shortcut_second_arc`) contient l'id du premier demi-edge du shortcut (un shortcut est l'agrégat de DEUX edges) -> celui-ci peut-être réel ou virtuel.

Important : tous les edges du graphe original ne sont pas présents dans `ch.forward` et `ch.backward` ! Seuls ceux qui contribuent à un plus court-chemin (ou plus exactement, dont on n'a pas pu prouver qu'ils n'y contribuaient pas en trouvant un witness-path) seront dans `ch.forward` et `ch.backward`. (EDIT : en fait, je pense que cette phrase est erronée : chaque edge original est au moins ajouté à `ch.forward` ou `ch.backward` au moment de la contraction du plus petit de ses nodes).

Loup : `shortcut_first_arc` et `shortcut_second_arc` renvoient l'id du demi-edge du shortcut, mais cet id **n'est pas à utiliser de la même façon** selon qu'on parle du premier ou deuxième demi-edge. Si on parle d'un shortcut de `ch.forward` :
-  on a vu plus haut que le middle-node `X` avait le rank le plus bas : `X < A < B`
-  le **SECOND** demi-edge `XB` est donc un edge **FORWARD** (et l'index renvoyé par `shortcut_second_arc` est donc un index dans `ch.forward.head` / `ch.forward.weight` / ...)
-  de même, le **PREMIER** demi-edge est un edge **BACKWARD** (et l'index renvoyé par `shortcut_first_arc` est donc un index dans `ch.backward.head` / `ch.backward.weight` / ...)

Cas particulier : lorsqu'un edge `E` de `ch.forward` (ou `ch.backward`) est un edge **original**, on peut le retrouver dans les structures passées en entrée de `ContractionHierarchy::build`, à savoir `tail`, `head` et `weight` :
-  `shortcut_first_arc` contient alors l'index `i` de l'edge original `E` dans `tail`, `head`, et `weight`
-  `shortcut_second_arc` contient alors l'index du tail de E (i.e. on devrait toujours avoir `tail[shortcut_first_arc[E]] == shortcut_second_arc[E]`)
-  (à noter que ceci est vrai si on parle d'un edge `E` dans `ch.forward`, si on parle d'un `E` dans le graphe `ch.backward`, alors ce sera inversé, et c'est dans `head` qu'on retrouvera le node)

### Bug probable

Il y a (selon moi) un bug dans l'implémentation de RoutingKit :
- il y a lorsqu'on s'intéresse à un edge (original) du graphe `ch.forward`, son `shortcut_second_arc` ne contient pas son **tailnode** comme il le devrait d'après [ce commentaire](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/include/routingkit/contraction_hierarchy.h#L60), mais il contient (à tort son `headnode`). L'origine du bug est dans [cette ligne](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1028) :

```cpp
// CODE ACTUEL (buggé) :
ch.backward.shortcut_second_arc[xy] = head[a];

// CODE CORRECT :
ch.backward.shortcut_second_arc[xy] = tail[a];
```
- la ligne serait correcte si on était dans `ch.backward` (c'est le cas juste au dessus), donc c'est sans doute une erreur de copié-collé.
- ce bug reste à confirmer, car je trouve curieux dans ce cas que l'unpacking forward fonctionne correctement, vu que [cette ligne](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1728) utilise le code buggé...

**EDIT** : après essais exhaustifs sur un graphe de test minimaliste pour comprendre ce que contient `shortcut_second_arc`, je pense avoir compris, en résumé, ce n'est pas le [code d'attribution du head](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1028) qui était buggé, mais plutôt [le commentaire](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/include/routingkit/contraction_hierarchy.h#L60) qui était faux, et qui ne s'applique qu'aux shortcuts originaux backward. J'y reviens ci-dessous.


## Construction de shortcut_first_arc / shortcut_second_arc

Ce sont les structures `shortcut_first_arc` et `shortcut_second_arc` qui permettent d'unpack un shortcut, jusqu'à remonter récursivement aux demi-edges originaux.

En pratique, c'est la fonction `build_unpacking_information` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L973)) qui construit ces structures, en utilisant `ContractionHierarchyExtraInfo` pour avoir connaissance du mid-node (sans quoi on dirait qu'on n'y a pas accès dans les structures de `ContractionHierarchy`) ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1002)).

[Cette ligne](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1014) me donne confirmation de ma compréhension des choses jusqu'ici :

```cpp
assert(ch.forward.weight[xy] == ch.backward.weight[ch.forward.shortcut_first_arc[xy]] + ch.forward.weight[ch.forward.shortcut_second_arc[xy]]);
```

Interprétation : on s'intéresse au shortcut `xy` dans le graphe `ch.forward`. Son premier demi-edge est dans le graphe `ch.backward`, et son second demi-edge est dans le graphe `ch.forward`, c'est cohérent avec ce qui est décrit plus haut.

## Comment unpacker un shortcut ?

Avec ma compréhension toute fraîche, je peux expliquer le code d'unpacking, exemple avec `get_arc_path` ([lien](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1760)) :

- **STEP 1** = à partir du meeting node, on remonte de proche en proche en arrière (jusqu'à la source du trajet, donc) dans le forward-path, en récupérant à chaque fois le prédécesseur de chaque node, pour reconstruire le forward-path complet jusqu'au meeting-node :

    ```cpp
    std::vector<unsigned>up_path;
    unsigned x = shortest_path_meeting_node;
    while(forward_predecessor_node[x] != invalid_id){
        up_path.push_back(forward_predecessor_arc[x]);
        x = forward_predecessor_node[x];
    }
    ```
- à l'issue de cette STEP 1, on dispose du forward-path **contracté**, sous forme d'une succession de nodes (`vector<unsigned>`), dans l'ordre inverse du parcours. Les arcs qui forment ce forward-path sont des **SHORTCUTS**, qu'il faut unpacker.
- **STEP2** = en les parcourant à l'envers (pour remettre le forward-path dans le bon sens), on unpack chaque arc :
    ```cpp
    for(unsigned i=up_path.size(); i>0; --i){
        unpack_forward_arc(*ch, up_path[i-1], [&](unsigned xy, unsigned y){path.push_back(xy);});
    }
    ```
- la fonction `unpack_forward_arc` est une fonction récursive ([lien](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1726)) qui soit appelle `on_new_input_arc` si l'arc est un arc original, soit s'appelle récursivement sur ses deux demi-edges si l'arc est un shortcut :
    ```cpp
    if(ch.forward.is_shortcut_an_original_arc.is_set(arc)){
        on_new_input_arc(ch.forward.shortcut_first_arc[arc], ch.forward.shortcut_second_arc[arc]);
    } else {
        unpack_backward_arc(ch, ch.forward.shortcut_first_arc[arc], on_new_input_arc);
        unpack_forward_arc(ch, ch.forward.shortcut_second_arc[arc], on_new_input_arc);
    }
    ```
- **LE POINT TRÈS IMPORTANT** : on a besoin de traiter différemment le premier et le second demi-edge d'un shortcut (cf. plus haut), d'où le fait qu'on ait deux fonctions :
    * `unpack_backward_arc` pour unpacker le **premier** demi-edge (et qui va utiliser `ch.backward`), [lien](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1726)
    * `unpack_forward_arc` pour unpacker le **second** demi-edge (et qui va utiliser `ch.forward`), [lien](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1738)
- à noter que `on_new_input_arc` se contente d'ajouter l'arc au chemin (le fait qu'on utilise cette fonction sous forme d'une callback n'est là que pour mutualiser le code avec `get_node_path`)
- **STEP3** = on fait de même avec le backward-path ([lien](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1780))
- à l'issue de toutes ces étapes, on a reconstitué le path, qui est maintenant constitué d'edges **réels** (manipulés par leurs ids dans le graphe ORIGINAL passé à `ContractionHierarchy::build`)

En résumé :
- on reconstitue le chemin contracté (à partir des structures qu'a rempli le dijkstra bidirectionnel, notamment `shortest_path_meeting_node`, `forward_predecessor_node`, `forward_predecessor_arc`, etc.)
- on expande chaque edge-contracté, en expandant récursivement ses deux demi-edges
- pour expand chaque demi-edge, on utilise `ch.backward` pour le premier, et `ch.forward` pour le second

## Comment unpacker un shortcut — compléments sur `shortcut_second_arc`

Rappel du problème : comment unpacker un shortcut = si j'ai un index de shortcut dans un graphe (forward ou backward), comment le transformer en une succession d'edges originaux, ou de leurs nodes ?

NOTE : cette section analyse plus finement l'utilisation de `shortcut_second_arc` pour unpacker les shortcuts. Notamment, dans les structures d'une CH, quand on a un edge (par exemple, quand on a son index = un offset dans les arrays), on a facilement accès à son head, mais pas à son tail, car c'est à plutôt à partir du tail et du `first_out` qu'on a récupéré l'index de l'edge. Or, les deux demi-edges d'un shortcut `A → B` _partent_ du middle-node (car il a le rank le plus faible), du coup, la question est : comment trouver leur tail ? Comment trouver le middle-node du shortcut `A → B` ?

Spoiler alert = la réponse est "le middle-node sera le tail d'un edge original backward, donc stocké dans le `shortcut_second_arc` d'un demi-edge".

Préambule = il y a deux trucs contre-intuitifs nécessaires à connaître avant de pouvoir unpacker :

- d'abord, les dénominations `first` et `second` dans `shortcut_first/second_arc` font référence au premier et deuxième demi-edges DANS LE SENS DU GRAPHE ORIGINAL ! Dit autrement, si, sur le graphe original, on a 3 nodes consécutifs `A → X → B`, avec `X` le plus petit des trois :
    - dans tous les cas, `X → B` sera un edge (original) dans le graphe FORWARD, et `X → A` sera un edge (original ausis) dans le graphe BACKWARD
    - si `A < B`, on aura le shortcut `A → B` dans le graphe FORWARD. Dans ce cas, `shortcut_first_arc` pointera sur `X → A` dans le graphe BACKWARD, et `shortcut_second_arc` pointera sur `X → B` dans le graphe FORWARD
    - si `A > B`, on aura le shortcut `B → A` dans le graphe BACKWARD. Dans ce cas, `shortcut_first_arc` pointera... quand même `X → A` dans le graphe BACKWARD ! Et `shortcut_second_arc` pointera aussi sur `X → B` dans le graphe FORWARD
    - ce qui est contre-intuitif, c'est que selon que le shortcut est forward ou backward, son `shortcut_second_arc` pointera vers l'edge à destination du head du shortcut (si shortcut forward) ou de son tail (si shortcut backward).
- puis, [ce commentaire](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/include/routingkit/contraction_hierarchy.h#L60) n'est valable **QUE** pour les edges backward ! Le commentaire dit que pour les edges originaux, `shortcut_second_arc` pointe vers le tail-node de l'edge, mais en réalité :
    - si l'edge original est dans le graphe BACKWARD, alors `shortcut_second_arc` pointe bien vers le tail-node de l'edge
    - si l'edge original est dans le graphe FORWARD, alors `shortcut_second_arc` pointe en fait vers le HEAD de l'edge !

Même si le commentaire est faux, le mécanisme réellement implémenté est en fait bien pratique, puisqu'il permet l'unpacking récursif des shorcuts (cf. exemples concrets ci-après) :

- si un edge est original, renvoyer le contenu de `shortcut_second_arc` (qui pointera vers son head-node ou son tail-node, selon que l'edge était forward ou backward)
- si un edge est un shortcut, renvoyer la concaténation de l'unpacking de son premier demi-edge et de son deuxième demi-edge

Au final, avec ce mécanisme, l'unpacking d'un shortcut `A → X → Y → Z → B` renvoie tous les nodes sauf le premier : `X → Y → Z → B`.

Voici un premier exemple concret unpackant un shortcut FORWARD : supposons le shortcut `A → (X) → B` (avec `B > A`) qui passe par `X`, et dont les deux demi-edges `A → X` et `X → B` sont originaux :

- le shortcut `A → B` est présent dans le graphe FORWARD, de middle-node `X` (mais ça on ne le sait pas encore)
- son `shortcut_first_arc` pointe vers son premier demi-edge dans le sens original, soit `A → X` ; il est stocké sous sa forme inversée `h1 = X → A` dans le graphe BACKWARD.
- son `shortcut_second_arc` pointe vers son deuxième demi-edge dans le sens original, soit `X → B` ; il est stocké sous sa forme directe `h2 = X → B` dans le graphe FORWARD.
- on voit donc que les deux demi-edges PARTENT du middle-node `X`pour rejoindre les extrémités `A` et `B` du shortcut (`h1` en backward, `h2`en forward).
- pour unpacker `A → B`, on unpacke donc récursivement `h1` et `h2`, dans cet ordre, puis on concatène leur résultat :
    - `h1` est un edge original, donc on renvoie son `shortcut_second_arc`
    - comme `h1` est un edge BACKWARD, son `shortcut_second_arc` contient son tail-node
    - le tail-node de `h1 = X → A` est `X`, donc unpacker `h1` nous renvoie `X`
    - on passe à `h2`...
    - `h2` est un edge original, donc on renvoie son `shortcut_second_arc`
    - comme `h2` est un edge FORWARD, son `shortcut_second_arc` contient son head-node
    - le tail-node de `h2 = X → B` est `B`, donc unpacker `h2` nous renvoie `B`
- la concaténation de l'unpacking de `h1` et `h2` est donc `X → B`, donc unpacker `A → X → B` nous renvoie bien `X → B`, comme souhaité

Voici un deuxième exemple concret similaire au premier, mais pour un shortcut BACKWARD : supposons le shortcut `A → (X) → B` (avec `A > B`) qui passe par `X`, et dont les deux demi-edges `A → X` et `X → B` sont originaux :

- le shortcut `A → B` est en réalité présent dans le graphe BACKWARD sous sa forme inversée `B → A`, de middle-node `X` (mais ça on ne le sait pas encore)
- son `shortcut_first_arc` pointe vers son premier demi-edge **dans le sens original**, soit `A → X` ; il est stocké sous sa forme inversée `h1 = X → A` dans le graphe BACKWARD.
- son `shortcut_second_arc` pointe vers son deuxième demi-edge **dans le sens original**, soit `X → B` ; il est stocké sous sa forme directe `h2 = X → B` dans le graphe FORWARD.
- on voit donc que les deux demi-edges PARTENT du middle-node `X`pour rejoindre les extrémités `A` et `B` du shortcut (`h1` en backward, `h2`en forward).
- pour unpacker `B → A`, on unpacke donc récursivement `h1` et `h2`, dans cet ordre, puis on concatène leur résultat :
    - `h1` est un edge original, donc on renvoie son `shortcut_second_arc`
    - comme `h1` est un edge BACKWARD, son `shortcut_second_arc` contient son tail-node
    - le tail-node de `h1 = X → A` est `X`, donc unpacker `h1` nous renvoie `X`
    - on passe à `h2`...
    - `h2` est un edge original, donc on renvoie son `shortcut_second_arc`
    - comme `h2` est un edge FORWARD, son `shortcut_second_arc` contient son head-node
    - le tail-node de `h2 = X → B` est `B`, donc unpacker `h2` nous renvoie `B`
- la concaténation de l'unpacking de `h1` et `h2` est donc `X → B`, donc unpacker `A → X → B` nous renvoie bien `X → B`, comme souhaité

On vient de voir un truc très important : unpacker un shortcut backward donne bien sa succession de nodes originaux DANS L'ORDRE DU GRAPHE ORIGINAL (même si l'edge unpacké était inversé, on reçoit les nodes dans l'ordre original !).

En fait, "grâce" au truc contre-intuitif n°1, l'unpacking du shortcut original `A → (X) → B` suivra exactement les mêmes étapes, peu importe que `A > B` ou `B < A`.

Troisième et dernier exemple concret : supposons le shortcut `A → (X) → (Y) → B`, avec `B > A` (donc le shortcut est forward), et `X < Y` (donc le demi-edge `A → Y` est lui-même un shortcut, passant par `X`) :

- le shortcut `A → B` est présent dans le graphe FORWARD, de middle-node `Y` (mais ça on ne le sait pas encore)
- son `shortcut_first_arc` pointe vers son premier demi-edge dans le sens original, soit `A → Y` ; il est stocké sous sa forme inversée `h1 = Y → A` dans le graphe BACKWARD.
- son `shortcut_second_arc` pointe vers son deuxième demi-edge dans le sens original, soit `Y → B` ; il est stocké sous sa forme directe `h2 = Y → B` dans le graphe FORWARD.
- pour unpacker `A → B`, on unpacke donc récursivement `h1` et `h2`, dans cet ordre, puis on concatène leur résultat.
- `h1` est lui aussi un **SHORTCUT**, contitué de deux quarts d'edge. Même si `h1` est un shortcut du graphe BACKWARD, ses deux quarts d'edges sont à unpacker dans le sens ORIGINAL (conformément au truc contre-intuitif) :
    - son `shortcut_first_arc` pointera vers `A → X`, qui sera stocké sous sa forme inversée `q1 = X → A` dans le graphe BACKWARD
    - son `shortcut_second_arc` pointera vers `X → Y`, qui sera stocké sous sa forme directe `q2 = X → Y` dans le graphe FORWARD
- l'unpacking de `A → B` sera donc la concaténation dans cet ordre de l'unpacking de ces 3 shortcuts :
    - `q1 = X → A` dans le graphe BACKWARD
    - `q2 = X → Y` dans le graphe FORWARD
    - `h2 = Y → B` dans le graphe FORWARD
- comme tous ces edges sont originaux, on renvoie ce que retourne leur `shortcut_second_arc` (soit le tail pour un edge BACKWARD, et le head pour un forward), l'unpacking sera donc :
    - `q1 = X → A` dans le graphe BACKWARD sera unpacké en `X`
    - `q2 = X → Y` dans le graphe FORWARD sera unpacké en `Y`
    - `h2 = Y → B` dans le graphe FORWARD sera unpacké en `B`
- au final, l'unpacking de `A → B` est donc `X → Y → B`, comme souhaité :-)


Pour revenir sur le "bug" (en réalité, plutôt un commentaire qui ne s'applique qu'au backward) : il aurait été possible de "corriger" le bug en faisant en sorte que le `shortcut_second_arc` des demi-edges forward originaux pointent également sur leur tail (donc sur le middle-node), mais ça aurait rendu le code d'unpacking moins élégant :
- actuellement : l'unpacking est élégant = si l'edge est un shortcut, on récurse sur ses deux demi-edges ; sinon, on renvoie le node contenu dans son `shortcut_second_arc`
- si on "corrigeait" le bug : l'unpacking serait moins élégant = si l'edge est un shortcut, on récurse sur ses deux demi-edges ; sinon, on renvoie soit son head-node si l'edge est forward, soit le node contenu dans son `shortcut_second_arc` si l'edge est backward
