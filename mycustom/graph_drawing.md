# Dessin d'un graphe

## Contexte

L'objectif est de pouvoir dumper les arêtes et nodes d'un graphe dans une image.

## Notes vrac

**TL;DR** : le dump d'un graphe SVG est rudimentaire, insuffisant pour mes besoins d'illustration.

Concernant la géométrie, la doc ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/OpenStreetMap.md)) indique explicitement qu'elle n'est pas incluse pour le moment, mais qu'un contournement est de considérer que chaque paire de noeuds dans la donnée OSM est un edge : le dump SVG de chaque edge revient à dumper la géométrie exacte des edges, mais au prix d'un graphe qui ne représente plus la réalité du routing.

Il faut passer un booléen au chargement du graphe :

```cpp
bool CONSIDER_THAT_MODELLING_NODES_ARE_ROUTING_NODES = true;
auto graph = simple_load_osm_pedestrian_routing_graph_from_pbf(
    osm_file,
    [](string const&){},
    CONSIDER_THAT_MODELLING_NODES_ARE_ROUTING_NODES
);
```

Par ailleurs, le binaire qui dump le SVG se contente de tracer des lignes assez brutes entre deux noeuds, et on n'a donc pas un joli chemin SVG.
