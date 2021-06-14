# Rank des différents nodes lors de la contraction d'un node `X`

On s'intéresse aux nodes `A → X → B` lors de la contraction du node `X`.

Dit autrement, `X` a donc un **in-edge** depuis `A`, et un **out-edge** vers `B`.

**ATTENTION** : ne pas confondre :
- **le sens de parcours** de l'edge. Dans notre exemple, partant de `A`, on ne peut aller que VERS `X`, puis VERS `B` (dit autrement : dans la vraie vie, il n'existe pas de route allant de `B` vers `X`).
- **les ranks relatifs** de `A`, `X` et `B` ; dans notre exemple, on pourra avoir indifféremment `A > B` ou `B > A`, sans lien avec le sens de parcours (par contre, comme on contracte `X` en premier, il aura **toujours** un rank plus petit que `A` et `B`).

## Préambule = où la notion de "shortcut" a-t-elle du sens ?

Un shortcut (i.e. un edge entre deux noeuds, inexistant dans le graphe original) n'existe QUE dans ces deux contextes :
- soit dans un graphe **Side** (`ch.forward` ou `ch.backward`), graphes pérennes sur lesquels s'effectuera plus tard la propagation
- soit dans **le graphe de contraction**, appelé `Graph` dans le code, graphe transitoire qui ne sert PAS à la propagation

Dans le graphe de contraction, il existera temporairement un shortcut `A → B` : celui-ci est ajouté par la contraction de `X`, et sera supprimé par la contraction de `A` ou de `B`.

Dans ce graphe de contraction, on pourra donc très bien avoir (temporairement, donc) un shortcut de `A` vers `B`, même si `A > B`, donc avoir un shortcut dans le "mauvais" sens, i.e. le sens "descendant les ranks". Ça n'est PAS grave : ce n'est pas sur ce **graphe de contraction** que la propagation va être faite, mais sur les graphe **Side** ! Or, c'est à la propagation qu'on n'a le droit d'aller que vers des nodes de rank supérieur : le graphe de contraction "a le droit" d'avoir des shortcuts dans le mauvais sens.

Et on va montrer que dans les deux cas (`A < B` ou `A > B`), dans les graphe **Side** `ch.forward` et `ch.backward`, l'ordering des nodes est respecté.

## Cas 1 = A > B

- juste avant de contracter `X`, on ajoute l'out-edge `XB` à `ch.forward` (qui respecte l'ordering des nodes, puisque `X < B`)
- de même, juste avant de contracter `X`, on ajoute l'in-edge `AX` à `ch.backward` (dans ce graphe, il devient donc `XA`, ce qui respecte l'ordering des nodes, puisque `X < A`)
- **conclusion** : si `A > B`, les edges de `X` ajoutés à `ch.forward` et `ch.backward` sont dans le bon sens = toujours vers le node de rank supérieur.
- une fois les graphes **Side** mis à jour, on contracte `X`, ce qui modifie le **graphe de contraction**, on lui ajoute un edge shortcut `A → B`
- ce shortcut est dans le "mauvais" sens (car `A > B`, donc il se dirige vers le rank inférieur), mais comme dit plus haut, CE N'EST PAS GRAVE dans le **graphe de contraction**
- plus tard, comme `A > B`, `B` sera le prochain nodes des trois à être contracté
- juste avant de contracter `B`, le shortcut `AB` du graphe de contraction (qui est alors vu comme un in-edge de `B`) sera ajouté à `ch.backward` en inversé, donc on ajoute `BA` à **ch.backward**, ce qui respecte bien l'ordering des nodes, puisque `A > B` !

## Cas 2 = B > A

Laissé en exercice au lecteur ;-)

## En résumé :

- comme c'est lui qu'on contracte avant `A` et `B`, le mid-node `X` a le rank le plus petit
- peu importe l'ordering relatif de `A` par rapport à `B`, si `X` est contracté en premier, alors on aura l'edge `XB` dans le graphe `ch.forward`, et `XA` dans le graphe `ch.backward`, ce qui respecte bien les ranks des nodes.
- selon que `A > B` ou non, on aura le shortcut `AB` dans le graphe `ch.forward`, ou le shortcut `BA` dans le graphe `ch.backward`, mais dans les deux cas, les ranks des nodes seront respectés.
- dans le graphe de contraction, il existera (de façon transitoire) le shortcut `AB` (qui sera dans le mauvais sens si `A > B`, et ce n'est pas grave)

Une conséquence à comprendre :
- si on trouve un shortcut `AB` dans le graphe `ch.forward`, alors on avait forcément `X < A < B`  (et le sens de parcours était `A → X → B`), le premier demi-edge `AX` (ou plutôt `XA`) dans le graphe `ch.backward`, et le second demi-edge `XB` est dans `ch.forward`.
- si on trouve un shortcut `EF` dans le graphe `ch.backward`, alors on avait forcément `X < E < F` (et le sens de parcours était `F → X → E`), le premier demi-edge `FX` (ou plutôt `XF`) dans le graphe `ch.backward`, et le second demi-edge `XE` est dans `ch.forward`.
- dans tous les cas, les deux demi-edges sont dans des graphes **Side** différents !
