# Analyse du code CH — pré-requis

Ces notes correspondent aux investigations sur les structures utilisées par le code de contraction.

Elles ne concernent **pas** directement le code de contraction, mais elles sont tout de même nécessaires pour le comprendre.

* [Contexte et grosses mailles](#contexte-et-grosses-mailles)
  * [Du point de départ à l'appel de build_ch_and_order](#du-point-de-départ-à-lappel-de-build_ch_and_order)
  * [Classe ContractionHierarchy](#classe-contractionhierarchy)
  * [Classe ContractionHierarchyExtraInfo](#classe-contractionhierarchyextrainfo)
* [Structure Graph::Arc](#structure-grapharc)
* [Structure Graph](#structure-graph)
* [Classe ShorterPathTest](#classe-shorterpathtest)
* [notion = hops](#notion--hops)
* [Fonction estimate_node_importance](#fonction-estimate_node_importance)
* [Notions : order, rank, et inverse permutation](#notions--order-rank-et-inverse-permutation)
  * [TL;DR](#tldr)
  * [rank et order](#rank-et-order)
  * [Notion d'inverse permutation](#notion-dinverse-permutation)


## Contexte et grosses mailles

### Du point de départ à l'appel de `build_ch_and_order`

Je m'intéresse à l'ordering/contraction en tirant le fil à partir de [l'exemple donné dans le README](https://github.com/phidra/RoutingKit/blob/a7db9cdabaadb2865fa6dd9b99906b616e679c3b/README.md) :

```cpp
auto ch = ContractionHierarchy::build(
    graph.node_count(),
    tail, graph.head,
    graph.travel_time
);
```

La contraction est donc réalisée en appelant la fonction `build`, fonction statique de la classe `ContractionHierarchy` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/contraction_hierarchy.h#L22)).

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

On reçoit donc le graphe sous la forme d'une *EdgeList* comportant trois vector, comportant N (=nb edges) items :
- `tail` représente le noeud **de départ** de l'edge ("node_from")
- `head` représente le noeud **d'arrivée** de l'edge ("node_to")
- `weight` représente le poids de l'edge (!)

NdM : j'aime beaucoup la dénomination **head** et **tail**, que je trouve plus courte et moins ambigües que node_from/node_to...

On builde une instance vide de `ContractionHierarchy`, et `ContractionHierarchyExtraInfo` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1103) :

```cpp
    ContractionHierarchy ch;
    ContractionHierarchyExtraInfo ch_extra;
```

On cleane un peu les edges en entrée [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1111), et on les trie :

```cpp
sort_arcs_and_remove_multi_and_loop_arcs(node_count, tail, head, weight, input_arc_id, log_message);
```

QUESTION : les arcs sont triés par quoi exactement ? (possiblement par {tail, head})

In fine, toute cette fonction `ContractionHierarchy::build` sert surtout à wrapper `build_ch_and_order`, qui fait réellement la contraction ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1116)).

### Classe `ContractionHierarchy`

Cette classe représente le résultat de la contraction :
- l'ordre de contraction des nodes (order et rank permettent de retrouver un node à partir de son rank et vice-versa)
- le graphe upward `G↑` (appelé `forward`) où chaque node ne contient que des out-edges vers des node de rank **supérieur**
- le graphe downward `G↓` (appelé `downward`) où chaque node ne contient que des out-edges vers des node de rank **inférieur**

Grâce à une instance de cette classe, la classe `ContractionHierarchyQuery` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/contraction_hierarchy.h#L104on)) peut répondre à des requêtes de calcul d'itinéraire. EDIT : en première approche, un simple dijkstra bidirectionnel suffit pour utiliser cette structure → c'est ce que j'ai fait en python dans glifov.

```cpp
class ContractionHierarchy{

    // [...]
    unsigned node_count()const{ return rank.size(); }

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
};
```

### Classe `ContractionHierarchyExtraInfo`

La classe `ContractionHierarchyExtraInfo` semble être simplement une structure qui stocke le mid-node (probablement le node responsable de la création d'un shortcut lorsqu'on le contracte) [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L599) :

```cpp
struct ContractionHierarchyExtraInfo{
    struct Side{
        std::vector<unsigned>mid_node;
        std::vector<unsigned>tail;
    };

    Side forward, backward;
};
```

À confirmer, mais je pense qu'elle est sortie à part de la class `ContractionHierarchy` pour ne pas encombrer cette dernière si on ne s'intéresse qu'au temps de parcours (et pas au chemin effectivement emprunté).

## Structure `Graph::Arc`

C'est une structure interne à `Graph`, qui représente un edge partant d'un node `N` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L180) :

```cpp
struct Arc{
	unsigned node;
	unsigned weight;
	unsigned hop_length;
	unsigned mid_node;
};
```

Mon interprétation :

- `node` de destination
- `weight` de l'arc
- `hop_length` : nombre d'edges réels que l'arc représente (`1` si l'edge n'est pas un shortcut, `>1` sinon)
- `mid_node` = sans doute le noeud contracté (initialisé à `invalid_id` → ça indique que les arcs initiaux sont des arcs "réels", non-issus de la contraction d'un node)

## Structure `Graph`

C'est la structure représentant le graphe en cours de contraction [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L83).

**POINT IMPORTANT** : cette structure est constamment modifiée au fur et à mesure de la contraction, elle évolue constamment (pour se voir ajouter des shortcuts et supprimer des edgs réels) au gré de l'avancée de la contraction des nodes.

Le graphe est représenté sous-forme d'une liste d'adjacence (ce sont ses seuls attributs), stockant les in-edges et les out-edges [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L226) :

```cpp
std::vector<std::vector<Arc>>out_, in_;
std::vector<unsigned>level_;
```

De plus, chaque noeud se voit également attribuer un `level` qui n'est pas encore clair (mais qui est initalisé à `0` [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L90)).

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

## Classe `ShorterPathTest`

C'est une classe (la définition est ici : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L230)) qui sert à calculer des plus courts chemins sur un graphe.

Son utilisation typique est de chercher les **witness-paths** d'un chemin `A → X → B`, où `X` est le node en cours de contraction :
- entre un `in-node` (`A` ci-dessus) et un `out-node` (`B`) du node `X` en cours de contraction
- en bypassant `X` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L328)), i.e. les PCC calculés ne doivent **PAS** passer par `X`
- par rapport à une longueur de référence qui est la somme des deux longueurs `A → X` + `B → X`

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

## notion = hops

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

## Fonction `estimate_node_importance`

FIXME : notes et notions à poursuivre...

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

APPEL 2 = après la contraction d'un node, appelée sur chaque voisin du node contracté : https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L714

```cpp
for(auto x:neighbor_list){
    // [...]
    unsigned new_key = estimate_node_importance(graph, shorter_path_test, x);
    unsigned old_key = queue.get_key(x);
    if(old_key < new_key)
        queue.increase_key({x, new_key});
    else if(old_key > new_key)
        queue.decrease_key({x, new_key});
}
```


## Notions : `order`, `rank`, et inverse permutation

### TL;DR

Ce que sont exactement `ch.order` et `ch.rank` :

Définition de rank = un node de rank 0 a été contracté en premier.

- `ch.rank` est un vector qui associe un node-id au rank (permet de répondre à la question "quel est le rank de tel node ?")
- `ch.order` est un vector qui associe un rank à un node-id (permet de répondre à la question "quel est le node qui possède tel rank ?")

`rank` et `order` sont inverse-permutation l'un de l'autre.


### rank et order

"Preuve" dans le code de la signification de `ch.rank` et `ch.order` donnés plus haut :
Concernant `rank` et `order`, l'algo ressemble à ça :

```cpp

ch.rank.resize(node_count);
ch.order.resize(node_count);

// contracted_node_count est le RANK (le premier noeud à être contracté à un rank=0) :
unsigned contracted_node_count = 0;

while(!queue.empty()){
    unsigned node_being_contracted = queue.pop().id;

    // ch.rank -> INDEX=node-id  VALUE=rankk
    // répond à la question "quel est le rank de tel node-id ?"
    ch.rank[node_being_contracted] = contracted_node_count;

    // ch.order -> INDEX=rank  VALUE=node-id
    // répond à la question "quel est le node qui a tel rank ?"
    ch.order[contracted_node_count] = node_being_contracted;

    // [...]

    contract_node(graph, shorter_path_test, node_being_contracted);

    // [...]

    ++contracted_node_count;
}
```

À noter que par la suite, on dirait que le rank est modifié (de façon iso en terme de calcul d'iti, j'imagine) pour être plus cache friendly, par la fonction `optimize_order_for_cache` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L902)) :

```cpp
std::vector<unsigned>new_order(node_count);
// remplit `new_order` petit à petit...
ch.rank = invert_permutation(new_order);
ch.order = std::move(new_order);
```

### Notion d'inverse permutation

J'ai l'impression qu'il s'agit de "retrouver" un objet à partir de sa propriété. Par exemple, si chaque node est ranké, je peux stocker :
- un vector V1 dont l'index est le node-id, et le contenu est son rank
- un vector V2 dont l'index est le rank, et le contenu est le node-id ayant ce rank

D'après ce que je comprends [de cette doc](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/SupportFunctions.md), alors V1 est la permutation inverse de V2 (et vice-versa) :

```cpp

// INDEX=node-id     VALUE=rank :
// répond à la question "quel est le rank de ce node ?"
vector<unsigned>v = {3,0,2,1};                  // 

// INDEX=rank     VALUE=node-id :
// répond à la question "quel est le node dont le rank est tel-rank ?"
vector<unsigned>inv_p = invert_permutation(v);  // INDEX=rank     VALUE=node-id
assert(inv_p[0] == 1);
assert(inv_p[1] == 3);
assert(inv_p[2] == 2);
assert(inv_p[3] == 0);
```

C'est corroboré par le fait que `rank` et `order` sont `invert_permutation` l'une de l'autre : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L964) :

```cpp
ch.rank = invert_permutation(new_order);
ch.order = std::move(new_order);
```
