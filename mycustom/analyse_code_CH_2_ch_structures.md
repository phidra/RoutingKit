* [Les structures de CH](#les-structures-de-ch)
   * [Préambule = identification des nodes dans les graphes ch.forward et ch.backward](#préambule--identification-des-nodes-dans-les-graphes-chforward-et-chbackward)
   * [Les structures de ch.forward / ch.backward](#les-structures-de-chforward--chbackward)
      * [Bug probable](#bug-probable)
   * [Construction de shortcut_first_arc / shortcut_second_arc](#construction-de-shortcut_first_arc--shortcut_second_arc)
   * [Comment unpacker un shortcut ?](#comment-unpacker-un-shortcut-)

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

Important : tous les edges du graphe original ne sont pas présents dans `ch.forward` et `ch.backward` ! Seuls ceux qui contribuent à un plus court-chemin (ou plus exactement, dont on n'a pas pu prouver qu'ils n'y contribuaient pas en trouvant un witness-path) seront dans `ch.forward` et `ch.backward`.

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
