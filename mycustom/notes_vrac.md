# Notes vrac

**NOTE** : ce fichier contient des notes en vrac. D'autres fichiers plus spécifiques contiennent les notes de sujets particuliers :

- l'analyse du code CH est décrite dans [ce fichier](./analyse_code_CH.md).
- la construction de graphe à partir d'un fichier OSM (`file.pbf`) est décrite dans [ce fichier](./pbf_reading.md)
- le fait dedessiner un graphe, i.e. construire une image (PNG, SVG, ...) à partir d'un graphe est décrite dans [ce fichier](./graph_drawing.md).

## Fichiers intéressants

Je constate que la plupart des fichiers de RoutingKit/src sont des binaires ! Les seuls fichiers qui sont "utilitaires" sont :

```sh
grep -L "int main" RoutingKit/src/*.cpp

bit_select.cpp                    # fonctions utilitaires de manipulation de buffers
bit_vector.cpp                    # self-documenting name
buffered_asynchronous_reader.cpp  # self-documenting name
contraction_hierarchy.cpp               # GROS fichier, implémentant CH
customizable_contraction_hierarchy.cpp  # GROS fichier, implémentant CCH
expect.cpp                        # fonctions utilitaires pour les assert
file_data_source.cpp              # semble être une abstraction d'un fichier
geo_position_to_node.cpp          #
graph_util.cpp                    #
id_mapper.cpp                     #
nested_dissection.cpp             #
osm_decoder.cpp                   #
osm_graph_builder.cpp             #
osm_profile.cpp                   #
osm_simple.cpp                    #
protobuf.cpp                      #
strongly_connected_component.cpp  #
timer.cpp                         #
vector_io.cpp                     #
verify.cpp                        #
```


Les fichiers importants sont plus visibles si je classe par taille de fichiers :

```sh
grep -L "int main" *.cpp | while read line
do
wc -l $line
done | sort -n

5    expect.cpp
18   timer.cpp
37   protobuf.cpp
64   vector_io.cpp
95   verify.cpp
99   strongly_connected_component.cpp
101  bit_select.cpp
129  id_mapper.cpp
155  osm_simple.cpp
167  graph_util.cpp
185  buffered_asynchronous_reader.cpp
188  file_data_source.cpp
232  geo_position_to_node.cpp
529  bit_vector.cpp
765  osm_decoder.cpp
801  osm_profile.cpp
830  osm_graph_builder.cpp
882  nested_dissection.cpp
1920 customizable_contraction_hierarchy.cpp
2217 contraction_hierarchy.cpp
```

## Généralités / vrac

- Doc intéressante = comment builder : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/Setup.md)
- Doc intéressante = contraction hierarchy : [lien](https://github.com/RoutingKit/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/ContractionHierarchy.md)
- Ils mentionnent le bucket-sort comme un algo de tri plus rapide pour trier les `int` : [lien wikipedia](https://en.wikipedia.org/wiki/Bucket_sort)
- Le build de la CH semble paramétré par un `max_pop_count`
- Au sujet de l'ordering :

    > A central component of the preprocessing consists of computing a so-called contraction order.
    > This is an ordering of the input nodes.
    > A significant fraction of the preprocessing running time is spent computing this order.
    > It can therefore be beneficial to store this ordering to accelerated the preprocessing.

- Il y a des fonctions pour sauvegarder l'ordering et la contraction sur le disque.
- La construction d'un query-object est lourde (et à recycler).
- Une fois la query traitée, l'objet expose des fonctions pour récupérer :
    - la distance entre source et target
    - le chemin sous forme d'une liste de nodes
    - le chemin sous forme d'une liste d'arcs
- On dirait que la license du code permet de réutiliser le code : [lien vers la LICENSE](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/LICENSE)
- La parallélisation peut-être parallélisée.
- On dirait que des customisations partielles (seulement quelques arcs) font l'objet d'une fonction (qui serait plus rapide ?), ça peut être utile : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/CustomizableContractionHierarchy.md#customizablecontractionhierarchypartialcustomization)
- Il y a toute une section dédiée au travail avec OSM, notamment il y a ce qu'il faut pour extraire facilement des graphes voiture/piéton : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/OpenStreetMap.md)

## À creuser un peu plus

- la fonction `sort_arcs_and_remove_multi_and_loop_arcs` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1111) , [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L17)
- la classe `MinIDQueue` [lien de l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L629), [lien de la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/id_queue.h) (c'est une pririty-queue qui stocke des ids (integer), en les classant selon un poids (appelé "key"), lui aussi entier. La fonction `pop` renvoie l'id qui a la plus **petite** key [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/id_queue.h#L78)
- ce que permet le travail avec les graphes OSM (et notamment, si je peux facilement conserver leur affichage géométrique), [lien vers la description](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/OpenStreetMap.md)
- l'affichage de graphe, car on dirait qu'il y a de quoi les dessiner au format SVG : [lien vers le binaire](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/graph_to_svg.cpp)  (EDIT : j'ai testé, c'est pas fi-fou... FIXME=pérenniser mes tests et leur conclusion)
- le stall-on-demand [lien vers la définition](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1536), [lien vers l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L1577)
- la notion de `level` d'un node contracté, [lien à l'utilisation](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L563), [lien de l'initialisation à 0](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L90), [lien de la modification du level des voisins d'un noeud fraîchement contracté](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L713)
- quel est l'intérêt de chercher à minimiser le hop-ratio ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L563)) ? (élément de réponse : ça cherche à minimiser la taille (en nombre de nodes et d'edges) du graphe "expanded", où on remplacerait chaque edge contracté par les edges réels dont il est constitué)
- plus généralement, mieux comprendre les subtilités de `estimate_node_importance` (avec des exemples concrets) me semble important pour maîtriser le sujet à fond
- la façon dont le `new_order` est calculé pour être plus cache-friendly, [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L902) (ça semble utiliser le `level`, justement)
- ce que fait `optimize_order_for_cache` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L902)), à l'oeil, on dirait qu'on réordonne les nodes pour "rapprocher" les nodes proches dans le graphe upward (resp. downard), de sorte que le parcours du graphe upward ait plus de chance de tomber sur des nodes déjà dans le cache ?
