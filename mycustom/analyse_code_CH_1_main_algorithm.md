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
