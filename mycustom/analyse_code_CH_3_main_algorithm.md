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

Derrière, tout l'algo consiste à traiter tous les noeuds du graphe l'un après l'autre, en respectant l'ordre donné par la queue (qui va évoluer au fur et à mesure de la contraction) [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L656) :

```cpp

while(!queue.empty()){
    unsigned node_being_contracted = queue.pop().id;
```

Notations :
- Soit `X` le node en cours de contraction (pour être cohérent avec mes notes précédentes), et `A → X → B` pour marquer les esprits.
- Dans ce qui suit, contraction-graph = le graphe de contraction (TODO = donner un lien + rappeler).

À la grosse louche, le corps de la boucle est :

- mémoriser les voisins de `X` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L662))
- ajouter les out-edges de `X` dans `ch.forward` (en effet, à ce stade, ils pointent tous vers des nodes qui auront un rank supérieur) ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L683)). **TAIL** = `X` (non-mémorisé par la structure `ch.forward`), **HEAD** = `B` = le head de l'out-edge.
- ajouter les in-edges de `X` dans `ch.backward` (en effet, à ce stade, ils proviennent tous de nodes qui auront un rank supérieur, donc l'edge inverse qu'on ajoute à `ch.backward` pointera vers un node de rank supérieur) ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L694)). **TAIL** = `X`, **HEAD** = `A` = le tail de l'in-edge `AX` (dit autrement : dans le graphe `ch.backward`, c'est en réalité l'edge **inversé** `XA` qui est stocké)
- contraction du node `X` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L709))
- plus en détail sur la contraction : ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L566)) :
    * on itère sur tous les triples `A → X → B` possibles : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L567)
    * s'il existe un autre plus court chemin de A à B que passant par X (un **witness-path**, on ne fait rien) : https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L574
    * sinon, c'est qu'il est nécessaire d'insérer un shortcut pour préserver le plus court chemin entre A et B à ce niveau de la hiérarchie
    * on le fait avec la fonction add_arc_or_reduce_arc_weight ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L579))
    * TODO = décrire cette fonction : https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L105
- mise à jour des voisins 1 = `raise_level` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L713))
- mise à jour des voisins 2 = update de la clé de queue  ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L714))

Point de détail (est-ce nécessaire de le metionner ?) = on ignore les boucles : https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L572

Pourquoi ça marche : l'invariant de boucle : au moment où on traite un node de la queue, il n'a d'out-edges que vers des nodes qui auront un ordre supérieur :
- les nodes traités précédemment ont été retirés du contraction-graph lors de leur contraction en fin de boucle
- les out-edges du noeud en cours de contraction pointent vers des nodes encore dans le contraction-graph
- les nodes encore dans le graphe seront traités après, donc auront un rank supérieur

TODO : continuer.
