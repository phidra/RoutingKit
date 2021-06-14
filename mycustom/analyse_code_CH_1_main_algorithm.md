# Algo principal CH

**FIXME** : ces notes sont encore à réorganiser.

**NOTE** : cette page s'intéresse à la fonction principale de contraction/ordering = `build_ch_and_order` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L608)). Elle suppose de connaître quelques prérequis : 

- comment on reçoit le graphe (tails, heads, weights)
- La classe ContractionHierarchy
- La classe ContractionHierarchyExtraInfo
- La structure Graph (notamment, il n'est pas figé, et est modifié au fur et à mesure de la contraction)
- La structure Graph::Arc
- La structure ShorterPathTest (notamment, le fait qu'il soit borné par max_pop_count)
- La notion de hops
- La notion `estimate_node_importance` , et ses critères
- Les notiosn d'order et rank


## fonction `build_ch_and_order`

**NOTE** : c'est cette fonction qui contient le coeur de l'algo à proprement parler.

Les paramètres d'appels [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L608) :

```cpp
void build_ch_and_order(
    Graph& graph,
    ContractionHierarchy& ch,
    ContractionHierarchyExtraInfo&ch_extra,
    unsigned max_pop_count,
    const std::function<void(std::string)>& log_message
)
```


### Déroulé de `build_ch_and_order`

On commence par ajouter dans la queue tous les noeuds du graphe, en estimant leur importance [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L632) :

```cpp
for(unsigned i=0; i<node_count; ++i) {
    queue.push({i, estimate_node_importance(graph, shorter_path_test, i)});
}
```

### Rank des différents nodes lors de la contraction d'un node `X`

On s'intéresse aux nodes `A → X → B` lors de la contraction du node `X`.

Dit autrement, `X` a donc un **in-edge** depuis `A`, et un **out-edge** vers `B`.

**ATTENTION** : ne pas confondre :
- **le sens de parcours** de l'edge. Dans notre exemple, partant de `A`, on ne peut aller que VERS `X`, puis VERS `B` (dans la vraie vie, il n'existe pas de route reliant `B` à `X`).
- **l'ordre relatif** de `A`, `X` et `B` ; dans notre exemple, on pourra avoir indifféremment `A > B` ou `B > A`, sans lien avec le sens de parcours (par contre, comme on contracte `X` en premier, il aura **toujours** un rank plus petit que `A` et `B`).

#### Préambule = où trouve-t-on des shortcuts ?

Un shortcut (i.e. un edge entre deux noeuds, inexistant dans le graphe original) n'existe QUE dans ces deux contextes :
- soit dans un graphe **Side** (`ch.forward` ou `ch.backward`), graphes pérennes sur lesquels s'effectuera plus tard la propagation
- soit dans **le graphe de contraction**, appelé `Graph` dans le code, graphe transitoire qui ne sert PAS à la propagation

Dans le graphe de contraction, il existera temporairement un shortcut `A → B` : celui-ci est ajouté par la contraction de `X`, et sera supprimé par la contraction de `A` ou de `B`.

Dans ce graphe de contraction, on pourra donc très bien avoir (temporairement, donc) un shortcut de `A` vers `B`, même si `A > B`, donc avoir un shortcut dans le "mauvais" sens, i.e. le sens "descendant les ranks". Ça n'est PAS grave : ce n'est pas sur ce **graphe de contraction** que la propagation va être faite, mais sur les graphe **Side** ! Or, c'est à la propagation qu'on n'a le droit d'aller que vers des nodes de rank supérieur : le graphe de contraction "a le droit" d'avoir des shortcuts dans le mauvais sens.

Et on va montrer que dans les deux cas (`A < B` ou `A > B`), dans les graphe **Side** `ch.forward` et `ch.backward`, l'ordering des nodes est respecté.

#### Cas 1 = A > B

**TODO** : on va montrer qu'il il n'existera aura AUCUN out-edge de A vers B. En effet, l'enchaînement est :

- juste avant de contracter `X`, on ajoute l'out-edge `XB` à `ch.forward` (qui respecte l'ordering des nodes, puisque `X < B`)
- de même, juste avant de contracter `X`, on ajoute l'in-edge `AX` à `ch.backward` (dans ce graphe, il devient donc `XA`, ce qui respecte l'ordering des nodes, puisque `X < A`)
- **conclusion** : si `A > B`, les edges de `X` ajoutés à `ch.forward` et `ch.backward` sont dans le bon sens = toujours vers le node de rank supérieur.
- une fois les graphes **Side** mis à jour, on contracte `X`, ce qui modifie le **graphe de contraction**, on lui ajoute un edge shortcut `A → B`
- ce shortcut est dans le "mauvais" sens (car `A > B`, donc il se dirige vers le rank inférieur), mais comme dit plus haut, CE N'EST PAS GRAVE dans le **graphe de contraction**
- plus tard, comme `A > B`, `B` sera le prochain nodes des trois à être contracté
- juste avant de contracter `B`, le shortcut `AB` du graphe de contraction (qui est alors vu comme un in-edge de `B`) sera ajouté à `ch.backward` en inversé, donc on ajoute `BA` à **ch.backward**, ce qui respecte bien l'ordering des nodes, puisque `A > B` !

#### Cas 2 = B > A

Laissé en exercice au lecteur ;-)

#### En résumé :

- comme c'est lui qu'on contracte avant `A` et `B`, le mid-node `X` a le rank le plus petit
- peu importe l'ordering relatif de `A` par rapport à `B`, si `X` est contracté en premier, alors on aura l'edge `XB` dans le graphe `ch.forward`, et `XA` dans le graphe `ch.backward`, ce qui respecte bien les ranks des nodes.
- selon que `A > B` ou non, on aura le shortcut `AB` dans le graphe `ch.forward`, ou le shortcut `BA` dans le graphe `ch.backward`, mais dans les deux cas, les ranks des nodes seront respectés.
- dans le graphe de contraction, il existera (de façon transitoire) le shortcut `AB` (qui sera dans le mauvais sens si `A > B`, et ce n'est pas grave)

Une conséquence à comprendre :
- si on trouve un shortcut `AB` dans le graphe `ch.forward`, alors on avait forcément `X < A < B`  (et le sens de parcours était `A → X → B`), `XB` est dans `ch.forward` et `AX` (ou plutôt `XA`) dans le graphe `ch.backward`.
- si on trouve un shortcut `EF` dans le graphe `ch.backward`, alors on avait forcément `X < E < F` (et le sens de parcours était `F → X → E`), `XE` est dans `ch.forward` et `FX` (ou plutôt `XF`) dans le graphe `ch.backward`.
- dans tous les cas, les deux demi-edges sont dans des graphes **Side** différents !

### NOTES VRAC À DISPATCHER

#### TO DISPATCH 1

La fonction `build_unpacking_information` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L973)), et le fait qu'on utilise `ContractionHierarchyExtraInfo` pour avoir le mid-node (sans quoi on dirait qu'on n'y a pas accès) ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1002)).

Le fonctionnement des shortcut_first_arc et shortcut_second_arc. [Cette ligne](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1014) donne des indication ssur comment l'utiliser :

```cpp
assert(ch.forward.weight[xy] == ch.backward.weight[ch.forward.shortcut_first_arc[xy]] + ch.forward.weight[ch.forward.shortcut_second_arc[xy]]);
```

Quand on y réfléchit, c'est logique : comme le mid-node d'un shortcut a le rank le plus petit des 3 nodes du shortcut, les deux edges du shortcut sont forcément l'un dans le graphe forward, l'autre dans le graphe backward.

Et même plus précisément, si on s'intéresse à un shortcut dans le graphe **forward**, ce shortcut est `A → X → B`, et comme on est sur le graphe forward, on a forcément `A < B`. De plus, comme `X` est le mid-node, il a été contracté avant `A` et `B`, il a donc le plus petit rank des 3, donc `X < A` et `X < B`.

On a donc forcément `X < A < B`.

Du coup, vus la relation d'ordre sur ces ranks, `AX` (premier arc du shortcut) sera dans le graphe **backward**, et `XB` (= deuxième arc du shortcut) sera dans le graphe **forward**. Ça cadre avec l'assert ci-dessus :-)


#### TO DISPATCH 2

Aha, on dirait que dans les "graphes" ch.forward et ch.backward, les nodes sont stockés par leur rank plutôt.

C'est assez logique, vu que c'est ça qui nous intéresse à la base pour savoir si on a le droit de les parcourir ; mais comme ça n'est PAS comme ça que ch.forward (resp. ch.backward) sont construits (en effet, ils sont construits avec les IDs de node, et non leur rank ! ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L689))), ça m'a confusé au début.

Il est probable qu'une fonction qui vient après `build_ch_and_order` post-processe la structure `ch` pour replacer les ids de nodes par leurs ranks. C'est sans doute l'une de ces 3 fonctions :

- `optimize_order_for_cache`
- `make_internal_nodes_and_rank_coincide`
- `sort_ch_arcs_and_build_first_out_arrays`


#### TO DISPATCH 3

J'ai pu vérifier que dans chaque graphe (forward ou backward) de la CH, on stocke des infos sur les EDGES :

- first_out contient N+1 éléments (où N = nombre de nodes), et référence les **edges** d'un node (cf. structure AdjacencyArray)
- derrière, TOUTES les autres structures contiennent E éléments (où E = nombre d'edges), un par edge dans le graphe CONTRACTÉ forward ou backward :
    * `head` est le noeud de destination de l'edge
    * `weight` est son poids
    * `is_shortcut_an_original_arc` (que j'appellerai plutôt "is_edge_an_original_arc")  est un booléen indiquant si l'edge dans `ch.forward` est réel (présent dans le graphe original) ou shortcut (issu de la contraction d'un node)
    * `shortcut_first_arc` (resp. second) contient l'id du premier edge du shortcut (un shortcut est l'agrégat de DEUX edges) -> celui-ci peut-être réel ou virtuel.

Quelques points importants en vrac :
- tous les edges du graphe original ne sont pas dans `ch.forward` et `ch.backward` ! Seuls ceux qui contribuent à un plus court-chemin (ou plus exactement, dont on n'a pas pu prouver qu'ils n'y contribuaient pas en trouvant un witness-path) seront dans `ch.forward` et `ch.backward`.
- lorsqu'on a un id d'edge renvoyé par `shortcut_first_arc` et `shortcut_second_arc`, ces ids sont les ids du premier ou du deuxième demi-edge du shortcut, et **ILS NE SONT PAS À UTILISER DE LA MÊME FAÇON** :
    * on a vu plus haut que le middle-node `X` avait le rank le plus bas : `X < A < B`
    * le **SECOND** demi-edge `XB` est donc un edge **FORWARD** (et l'index renvoyé par `shortcut_second_arc` est donc un index dans `ch.forward.head` / `ch.forward.weight` / ...)
    * de même, le **PREMIER** demi-edge `AX` est un edge **BACKWARD** (et l'index renvoyé par `shortcut_first_arc` est donc un index dans `ch.backward.head` / `ch.backward.weight` / ...)
- lorsqu'un edge `E` de `ch.forward` (ou `ch.backward`) est un edge **original**, on peut le retrouver dans les structures passées en entrée de `ContractionHierarchy::build`, à savoir `tail`, `head` et `weight` :
    * `shortcut_first_arc` contient alors l'index `i` de l'edge original `E` dans `tail`, `head`, et `weight`
    * `shortcut_second_arc` contient alors l'index du tail de E (i.e. on devrait toujours avoir `tail[shortcut_first_arc[E]] == shortcut_second_arc[E]`)
    * (à noter que ceci est vrai si on parle d'un edge `E` dans `ch.forward`, si on parle d'un `E` dans le graphe `ch.backward`, alors ce sera inversé, et c'est dans `head` qu'on retrouvera le node)
- (selon moi) il y a un bug dans l'implémentation de RoutingKit :
    * lorsqu'on s'intéresse à un edge (original) du graphe `ch.forward`, son `shortcut_second_arc` ne contient pas son **tailnode** comme il le devrait d'après [ce commentaire](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/include/routingkit/contraction_hierarchy.h#L60), mais il contient (à tort son `headnode`). L'origine du bug est dans [cette ligne](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1028) :

    ```cpp
    // CODE ACTUEL (buggé) :
    ch.backward.shortcut_second_arc[xy] = head[a];

    // CODE CORRECT :
    ch.backward.shortcut_second_arc[xy] = tail[a];
    ```
    * la ligne serait correcte si on était dans `ch.backward` (c'est le cas juste au dessus), donc c'est sans doute une erreur de copié-collé.
    * ce bug reste à confirmer, car je trouve curieux dans ce cas que l'unpacking forward fonctionne correctement, vu que [cette ligne](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1728) utilise le code buggé...

#### TO DISPATCH 4

Avec ma compréhension fraîche, je peux expliquer le code d'unpacking, exemple avec `get_arc_path` ([lien](https://github.com/RoutingKit/RoutingKit/blob/fb5e83bcd4cf85763fb6877a0b5f8d5736c9a15b/src/contraction_hierarchy.cpp#L1760)) :

- **STEP 1** = à partir du meeting node, on remonte de proche en proche en arrière (jusqu'à la source du trajet, donc) dans le forward-path, en récupérant à chaque fois le prédécesseur de chaque node :

    ```cpp
    std::vector<unsigned>up_path;
    unsigned x = shortest_path_meeting_node;
    while(forward_predecessor_node[x] != invalid_id){
        up_path.push_back(forward_predecessor_arc[x]);
        x = forward_predecessor_node[x];
    }
    ```
- on dispose maintenant du forward-path, sous forme d'une succession d'arcs, dans l'ordre inverse du parcours. Les arcs qui forment ce forward-path sont des **SHORTCUTS**, qu'il faut unpacker.
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
