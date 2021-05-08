# Fonctions de dessins

**Préambule** : notes vrac, à trier/organiser.

## Contexte

En gros, mon objectif, c'est de :
- charger un graphe à partir d'un fichier OSM (et idéalement d'un polygone limitant, encore que je peux utiliser la CLI osmium pour filtrer le PBF en amont)
- dessiner le graphe

## Notes vrac

Je pars du code du `main` indiqué dans le README [lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/README.md) :

```cpp
// Load a car routing graph from OpenStreetMap-based data
auto graph = simple_load_osm_car_routing_graph_from_pbf("file.pbf");
auto tail = invert_inverse_vector(graph.first_out);

// Build the shortest path index
auto ch = ContractionHierarchy::build(
    graph.node_count(),
    tail, graph.head,
    graph.travel_time
);
```

La fonction `simple_load_osm_car_routing_graph_from_pbf` renvoie un `SimpleOSMCarRoutingGraph` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/osm_simple.h#L30)) :


```cpp
SimpleOSMCarRoutingGraph simple_load_osm_car_routing_graph_from_pbf(
	const std::string&pbf_file,
	const std::function<void(const std::string&)>&log_message = nullptr,
	bool all_modelling_nodes_are_routing_nodes = false,
	bool file_is_ordered_even_though_file_header_says_that_it_is_unordered = false
);
```

Ce `SimpleOSMCarRoutingGraph` est une structure très simple agrégeant différents `vector` ([lien](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/include/routingkit/osm_simple.h#L10)) :

```cpp
struct SimpleOSMCarRoutingGraph{
	std::vector<unsigned>first_out;
	std::vector<unsigned>head;
	std::vector<unsigned>travel_time;
	std::vector<unsigned>geo_distance;
	std::vector<float>latitude;
	std::vector<float>longitude;
	std::vector<unsigned>forbidden_turn_from_arc;
	std::vector<unsigned>forbidden_turn_to_arc;
	unsigned node_count() const { return first_out.size()-1; }
	unsigned arc_count() const{ return head.size(); }
};
```

Un point intéressant : d'après [cette doc](https://github.com/phidra/RoutingKit/blob/a0776b234ac6e86d4255952ef60a6a9bf8d88f02/doc/SupportFunctions.md) l'utilisation de `first_out` et `head` indique que le graphe est stocké sous la forme d'un AdjacencyArray, et il y a donc une étape supplémentaire pour reconstruire le vector des `tail` :

```cpp
auto tail = invert_inverse_vector(graph.first_out);
```

