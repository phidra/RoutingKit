# Notes vrac

## Contexte

On s'intéresse à l'ordering/contraction en tirant le fil à partir de [l'exemple donné dans le README](https://github.com/phidra/RoutingKit/blob/a7db9cdabaadb2865fa6dd9b99906b616e679c3b/README.md) :

```cpp
auto ch = ContractionHierarchy::build(
    graph.node_count(),
    tail, graph.head,
    graph.travel_time
);
```

## Grosses mailles = fonction `ContractionHierarchy::build`

C'est une fonction statique de la classe `ContractionHierarchy` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/contraction_hierarchy.h#L22).

Quels sont ses paramètres d'appel [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1093) :

```cpp
ContractionHierarchy ContractionHierarchy::build(
    unsigned node_count,
    std::vector<unsigned>tail,
    std::vector<unsigned>head,
    std::vector<unsigned>weight,
    const std::function<void(std::string)>&log_message,  // default-value = fonction vide
    unsigned max_pop_count  // default-value = 500
)
```

On reçoit donc le graphe sous la forme d'une EdgeList comportant trois vector, comportant N (=nb edges) items :
- node_from = tail (int)
- node_to = head (int)
- weight (int)

NdM : j'aime beaucoup la dénomination **head** et **tail**, que je trouve plus courte et moins ambigües que node_from/node_to.

On builde une instance vide de la classe ContractionHierarchy, et ContractionHierarchyExtraInfo [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1103) :

```cpp
    ContractionHierarchy ch;
    ContractionHierarchyExtraInfo ch_extra;
```

La classe `ContractionHierarchyExtraInfo` semble être simplement une structure qui stocke le mid-node (probablement le node responsable de la création d'un shortcut lorsqu'on le contracte) [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L599).

On cleane un peu les edges en entrée [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1111), et on les trie :

```cpp
sort_arcs_and_remove_multi_and_loop_arcs(node_count, tail, head, weight, input_arc_id, log_message);
```

**QUESTION** : les arcs sont triés par quoi exactement ? (possiblement par {tail, head})

In fine, toute cette fonction sert surtout à wrapper `build_ch_and_order` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1116).


## fonction `build_ch_and_order`

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

Comprendre cette fonction nécessite de s'intéresser à d'autres structures.

Déroulé (to be continued) :


On commence par ajouter dans la queue tous les noeuds du graphe, en estimant leur importance [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L632) :

```cpp
for(unsigned i=0; i<node_count; ++i) {
    queue.push({i, estimate_node_importance(graph, shorter_path_test, i)});
}
```

### pré-requis = structure `Graph::Arc`

C'est une structure interne à `Graph`, qui représente un edge partant d'un node `N` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L180) :

- `node` de destination
- `weight` de l'arc
- `hop_length` (c'est quoi ? il est initialisé à `1` lorsqu'on créée le graphe → possiblement, c'est le nombre de contractions qu'il représente ?)
- `mid_node` = sans doute le noeud contracté (initialisé à invalid_id → ça indique que les arcs initiaux sont des arcs "réels", non-issus de la contraction d'un node)

### structure `Graph`

À confirmer en analysant, ça semble être la structure représentant le graphe en cours de contraction [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L83).

On dirait que le graphe est représenté sous-forme d'une liste d'adjacence (ce sont ses seuls attributs), stockant les in-edges et les out-edges [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L226) :

```cpp
std::vector<std::vector<Arc>>out_, in_;
std::vector<unsigned>level_;
```

De plus, chaque noeud se voit attribuer un `level` qui n'est pas encore clair (mais qui est iniitalisé à 0 [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L90)).


Son initialisation est assez straightforward = on itère sur chaque edge passé au contructeur, et on remplit les in-edges et out-edges de chaque noeud [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L92) :

```cpp
for(unsigned a=0; a<head.size(); ++a){
    unsigned x = tail[a];
    unsigned y = head[a];
    unsigned w = weight[a];


    if(x != y){
        out_[x].push_back({y, w, 1, invalid_id});
        in_[y].push_back({x, w, 1, invalid_id});
    }
}
```

## À creuser un peu plus

- la fonction `sort_arcs_and_remove_multi_and_loop_arcs` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1111) , [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L17)
- la classe `ShorterPathTest` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L625), [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L230) 
- la classe `MinIDQueue` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L629), [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/id_queue.h) (c'est une pririty-queue qui stocke des ids (integer), en les classant selon un poids (appelé "key"), lui aussi entier. La fonction `pop` renvoie l'id qui a la plus **petite** key [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/id_queue.h#L78)
- les permutations [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/permutation.h)
- la fonction `estimate_node_importance` [lien vers l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L633), [lien vers la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L527)
- `hop_length` [lien vers la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L183)
