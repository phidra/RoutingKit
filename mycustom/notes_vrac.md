# Notes vrac

**NOTE** : l'analyse du code CH est décrite dans [un autre fichier de notes](./analyse_code_CH.md).

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
- Notion d'inverse permutation ?
    - j'ai l'impression qu'il s'agit de "retrouver" un objet à partir de sa propriété
    - par exemple, si chaque node est ranké, je peux stocker :
        + un vector V1 dont l'index est le node, et le contenu est son rank
        + un vector V2 dont l'index est le rank, et le contenu est l'id du node ayant ce rank
    - d'après ce que je comprends [de cette doc](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/SupportFunctions.md), alors V1 est la permutation inverse de V2 (et vice-versa) :

    ```cpp
    vector<unsigned>p = {3,0,2,1};
    vector<unsigned>inv_p = invert_permutation(p);
    assert(inv_p[0] == 1);
    assert(inv_p[1] == 3);
    assert(inv_p[2] == 2);
    assert(inv_p[3] == 0);
    ```

    - c'est corroboré par le fait que `rank` et `order` sont `invert_permutation` l'une de l'autre : [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/src/contraction_hierarchy.cpp#L964) :

    ```cpp
    ch.rank = invert_permutation(new_order);
    ch.order = std::move(new_order);
    ```
