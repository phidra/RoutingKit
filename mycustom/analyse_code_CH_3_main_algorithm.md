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
    * on le fait avec la méthode `add_arc_or_reduce_arc_weight` du contraction-graph ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L579))
- plus de détails sur `add_arc_or_reduce_arc_weight` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L105) :
    * c'est une **méthode** du contraction-graph
    * si je garde mes notations, elle prend en entrée les 3 nodes `A → X → B`, et le poids (+hop_length) du raccourci `AB` qu'on va créer en contractant `X`
    * cas général = si l'arc `AB` n'existe pas encore dans le contraction-graph, on ajoute `AB` aux out-edges de `A`, et `AB` aux in-edges de `B`
    * si l'arc `AB` existe déjà dans le contraction-graph ET qu'il avait un poids supérieur, c'est que le shortcut qu'on veut ajouter est plus intéressant que l'edge pré-existant
    * dans ce cas, on ajoute le nouveau {weight + midnode + hop_length} en remplacement de ceux de l'edge `AB` préexistant
    * pas hyper-clair (mais pas bien grave) : si `AB` existe déjà dans le contraction-graph, selon qu'il y a plus d'in-edges qui arrivent en `X` ou au contraire d'out-edges qui en repartent, on réduit `AB` ou `BA` ...?
- mise à jour des voisins 1 = `raise_level` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L713))
- mise à jour des voisins 2 = update de la clé de queue  ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L714)). En gros, le score des voisins (qui reposait sur nombre de hops, nombre d'edges, et level) du node contracté a changé -> il faut le recalculer, ce qui aura pour effet possible de changer le plus petit élément du heap, donc de changer le prochain node à contracter.

Point de détail (est-ce nécessaire de le mentionner ?) = on ignore les boucles, i.e. les edges d'un noeud sur lui-même : https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L572

L'objectif = les propagation-graphs `ch.forward` et `ch.backward` ne doivent avoir QUE des edges vers les nodes de rank supérieur.

Pourquoi cette implémentation de la contraction marche ? L'invariant de boucle = au moment où on traite un node de la queue, il n'a d'out-edges que vers des nodes qui auront un ordre supérieur :
- les nodes traités précédemment ont été retirés du contraction-graph lors de leur contraction en fin de boucle
- les out-edges du noeud en cours de contraction pointent vers des nodes encore dans le contraction-graph
- les nodes encore dans le graphe seront traités après, donc auront un rank supérieur
