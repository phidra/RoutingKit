# Analyse du code CH

## Contexte

Je m'intéresse à l'ordering/contraction en tirant le fil à partir de [l'exemple donné dans le README](https://github.com/phidra/RoutingKit/blob/a7db9cdabaadb2865fa6dd9b99906b616e679c3b/README.md) :

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

Comprendre cette fonction nécessite de s'intéresser à d'autres structures.

FIXME : sortir ces notions préparatoires dans une doc à part.

## pré-requis = structure `Graph::Arc`

C'est une structure interne à `Graph`, qui représente un edge partant d'un node `N` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L180) :

- `node` de destination
- `weight` de l'arc
- `hop_length` (c'est quoi ? il est initialisé à `1` lorsqu'on créée le graphe → possiblement, c'est le nombre de contractions qu'il représente ?)
- `mid_node` = sans doute le noeud contracté (initialisé à invalid_id → ça indique que les arcs initiaux sont des arcs "réels", non-issus de la contraction d'un node)

## pré-requis = structure `Graph`

C'est la structure représentant le graphe en cours de contraction [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L83) ; en tant que telle, elle évolue constamment au gré de l'avancée de la contraction des nodes.

Le graphe est représenté sous-forme d'une liste d'adjacence (ce sont ses seuls attributs), stockant les in-edges et les out-edges [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L226) :

```cpp
std::vector<std::vector<Arc>>out_, in_;
std::vector<unsigned>level_;
```

De plus, chaque noeud se voit également attribuer un `level` qui n'est pas encore clair (mais qui est iniitalisé à 0 [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L90)).

L'initialisation de `Graph` est assez straightforward = on itère sur chaque edge passé au contructeur, et on remplit les in-edges et out-edges de chaque noeud [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L92) :

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

## pré-requis = `ShorterPathTest`

C'est une classe (la définition est ici : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L230)) qui sert à calculer des plus courts chemins sur un graphe.
Son utilisation typique est de chercher les *witness-paths* d'un `contracted-node` en cours de contraction :
- entre un `in-node` et un `out-node` du node en cours de contraction
- en bypassant le node en cours de contraction ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L328)), i.e. les PCC calculés ne doivent **PAS** passer par `contracted-node`
- par rapport à une longueur de référence (= la somme des deux longueurs `in-node > contracted-node` et `contracted-node > out-node`)

Son métier principal est donc sa méthode `does_shorter_or_equal_path_to_target_exist` (mais pour des raisons d'implémentation, le `in-node` et le `contracted-node` sont en fait passés à l'initialisation de la structure plutôt qu'à l'appel de cette méthode).

Exemple d'utilisation, réparti entre [sa CRÉATION](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L625), [son INITIALISATION](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L535), et l'[APPEL de sa méthode](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L533) :


```cpp

// CRÉATION de l'objet, qui va calculer des shortest path sur le graph (mutable)
ShorterPathTest shorter_path_test(graph, max_pop_count);

// [...]

// ici, `node` représente typiquement un node en cours de contraction.
// on itère sur ses in-edges :
for(unsigned in_arc = 0; in_arc < graph.in_deg(node); ++in_arc) {
    unsigned in_node = graph.in(node, in_arc).node;

    // INITIALISATION = on veut calculer des PCC qui partent de `in_node`
    //                  (et qui bypassent `node`, en cours de contraction)
    shorter_path_test.pin_source(in_node, node);
    for(unsigned out_arc = 0; out_arc < graph.out_deg(node); ++out_arc){
        unsigned out_node = graph.out(node, out_arc).node;
        if(in_node != out_node){
            if(
                // APPEL = on checke s'il existe un PCC (witness) entre la source et `out_node` :
                !shorter_path_test.does_shorter_or_equal_path_to_target_exist(
                    out_node,
                    graph.in(node, in_arc).weight + graph.out(node, out_arc).weight
                )
            ){
```

Détail d'implémentation : le comportement du calcul dépend d'un paramètre `max_pop_count` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L433) :

```cpp
if(pop_count > max_pop_count){
    return false;
}
```

En gros :
- `max_pop_count` semble correspondre au nombre de noeuds settled lors du dijkstra d'exploration qui recherche les PCC.
- C'est donc un paramètre permettant de borner la recherche.
- Si jamais on atteint cette limite sans avoir trouvé de PCC, `does_shorter_or_equal_path_to_target_exist` considère qu'il n'existait pas de PCC.
- La conséquence (acceptable) de cette approximation, c'est qu'on va insérer un shortcut qui était potentiellement inutile.

Point important : le graphe utilisé pour faire la recherche de PCC (`Graph`) peut (et va !) évoluer entre chaque utilisation. En effet, dans une utilisation typique, on alterne des appels à `ShorterPathTest.does_shorter_or_equal_path_to_target_exist` et à `contract_node`, et cette dernière fonction **modifie** la structure `Graph` sur laquelle travaille `ShorterPathTest`.


## pré-requis = hops

Ma compréhension est que les **hops** représentent le nombre d'edges "réels" (par opposition aux shortcuts, qui sont des edges "virtuels") :

- entre un noeud A et un noeud B reliés par un edge réel, il y a 1 edge `AB` et 1 hop
- entre un noeud A et un noeud B reliés par un shortcut (constitués de 2 edges réels, issus de la contraction de `V` entre `A` et `B`), il y a 1 edge `AB` réel, et **deux hops**


En effet, les `Arc` du graphe en cours de contraction (`Graph`) ont tous une propriété `hop_length` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L180) :

```cpp
struct Arc{
    unsigned node;
    unsigned weight;
    unsigned hop_length;
    unsigned mid_node;
};
```

Lors de leur initialisation (à partir du graphe non-contracté, sous forme d'une edge-list, donc à partir d'edges **réells**), ce `hop_length` est initalisé à `1` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L99) :

```cpp
out_[x].push_back({y, w, 1, invalid_id});
in_[y].push_back({x, w, 1, invalid_id});
```

Plus tard, les nouveaux edges ajoutés aux graphes  (i.e. les shortcuts) sont ajoutés avec un `hop_length` particulier, passé en paramètre de la fonction `add_arc_or_reduce_arc_weight` [lien 1](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L105), [lien2](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L151) :

```cpp
void add_arc_or_reduce_arc_weight(unsigned x, unsigned mid_node, unsigned y, unsigned weight, unsigned hop_length){
    // [...]
    out_[x].push_back({y,weight,hop_length,mid_node});
    in_[y].push_back({x,weight,hop_length,mid_node});
}
```

Et la valeur de ce `hop_length` particulier est la somme des `hop_length` des deux edges avant et après le node contracté [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L582) :

```cpp
graph.add_arc_or_reduce_arc_weight(
    // [...]
    graph.in(node_being_contracted, in_arc).hop_length + graph.out(node_being_contracted, out_arc).hop_length
);
```

## pré-requis = `estimate_node_importance`

À dire :

- fonction essentielle pour l'ordering
- sans doute l'une des fonctions responsable du plus de temps passé dans la contraction (ça sera intéressant de le vérifier, justement)
- appelée initialement pour initialiser les nodes
- appelée également après la contraction d'un node sur chacun de ses voisins, pour réévaluer lesdits voisins, suite à la contruction du node
- en gros, elle donne un score, et plus ce score sera bas, plus le node sera contracté rapidement
- le score est la somme de 3 facteurs qui ont le même poids
- facteur 1 = ratio entre le nombre de shortcuts ajoutés si on contracte le node et le nobmre d'edge supprimés par la contraction du node
- facteur 2 = ratio entre le nombre de hops ajoutés si on contracte le node et le nombre de hops supprimés par la contraction du node
- facteur 3 = le `level` du node en cours de contraction (pas encore très clair)

Ce qu'elle fait en gros :

DEFINITION
https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L527
unsigned estimate_node_importance(const Graph&graph, ShorterPathTest&shorter_path_test, unsigned node)

APPEL 1 = à l'initialisation du graphe
https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L633

```cpp
for(unsigned i=0; i<node_count; ++i) {
    queue.push({i, estimate_node_importance(graph, shorter_path_test, i)});
}
```


### Déroulé de `build_ch_and_order`

On commence par ajouter dans la queue tous les noeuds du graphe, en estimant leur importance [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L632) :

```cpp
for(unsigned i=0; i<node_count; ++i) {
    queue.push({i, estimate_node_importance(graph, shorter_path_test, i)});
}
```

## À creuser un peu plus

- la fonction `sort_arcs_and_remove_multi_and_loop_arcs` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1111) , [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L17)
- la classe `MinIDQueue` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L629), [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/id_queue.h) (c'est une pririty-queue qui stocke des ids (integer), en les classant selon un poids (appelé "key"), lui aussi entier. La fonction `pop` renvoie l'id qui a la plus **petite** key [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/id_queue.h#L78)
- les permutations [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/permutation.h)
- `hop_length` [lien vers la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L183)
- `max_pop_count` [lien vers l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1116)
- ce que sont exactement `ch.order` et `ch.rank` (note : la réponse est ici : [lien vers la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L659), le rank d'un node est le moment où il a été ordonné (i.e. un node de rank 0 aura été contracté en premier). L'order d'un rank est l'id du node de ce rank.
- ce que permet le travail avec les graphes OSM (et notamment, si je peux facilement conserver leur affichage géométrique), [lien vers la description](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/OpenStreetMap.md)
- l'afficahge de graphe, car on dirait qu'il y a de quoi les dessiner au format SVG : [lien vers le binaire](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/graph_to_svg.cpp)
- le stall-on-demand [lien vers la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1536), [lien vers l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1577)
- la notion de `level` d'un node contracté, [lien à l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L563), [lien de l'initialisation à 0](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L90), [lien de la modification du level des voisins d'un noeud fraîchement contracté](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L713)
