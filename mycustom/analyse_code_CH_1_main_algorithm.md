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

Et même plus précisément : comme le shortcut est `A → X → B`, on a forcément `A < B` (et `X` est le plus petit donc `X < A` et `X < B`), donc on a forcément `X < A < B`.

Du coup, `XB` (= deuxième arc du shortcut) sera dans le graphe **forward**, et `AX` (premier arc du shortcut) sera dans le graphe **backward**. Ça cadre avec l'assert ci-dessus :-)


