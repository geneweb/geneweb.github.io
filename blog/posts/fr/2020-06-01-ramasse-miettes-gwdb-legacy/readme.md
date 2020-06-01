# Un ramasse-miette pour GeneWeb

Jusqu'ici, supprimer définitivement un individu sans passer par un
export/import gw n'était pas possible. C'est déjà faisable, grâce à
l'outil `gwgc`.

## Préambule

GeneWeb, depuis peu, permet de paramétrer le moteur de stockage que
l'on souhaite utiliser pour enregistrer nos données. Cet article ne
concerne que le moteur de stockage historique (et au moment de
l'écriture le seul utilisable par GeneWeb). J'ai nommé `gwdb-legacy`.

## Un peu de technique

### Les personnes et les familles

Dans le moteur de stockage actuel de GeneWeb, les personnes, et les
familles sont stockées dans des tableaux.

Quand on supprime une personne, celle-ci n'est pas réellement
supprimée, mais la case qui contenait les données de la ressource que
l'on supprime est tout simplement vidée. En effet, les ressources sont
liées entre elles (liens de parenté, témoins aux évènements, ...) en
enregistrant, par exemple, le numero de la case correspondant aux
parents d'une personne dans le tableau. C'est pour cette raison, entre
autres, qu'on ne peut pas simplement *décaler* le tableau pour écraser
la case à supprimer.

On vide donc la case de toutes ses informations et on supprime les
liens vers/depuis cette case dans la base.

Cela crée donc en quelque sorte une nouvelle personne, vierge de toute
information, ce sont les fameuse personnes *? ?*.

Le problème, c'est que du coup cela créer des données fantôme, et
pollue la base de donnée. Il est possible, bien que ça ne soit pas ce
qu'il y a de plus intuitif ni de pratique, de recycler les personnes
fantômes en les re-remplissant au lieu d'ajouter une nouvelle
personne. Mais pour les familles, c'est impossible.

*Note: il serait enviseageable de recycler automatiquement les données
fantômes lors des nouvelles saisies, mais ce n'est pas le comportement
actuel de GeneWeb*

### Les chaînes de caractères (les "textes")

Les chaînes de caractères elles non plus ne sont pas réellement
supprimées. Pour des raisons techniques, elles sont, comme les
personnes et les familles, stockées dans un tableau. Lorsque l'on
enregistre le nom d'une personne, ce qui va etre enregistré dans la
personne, c'est en fait l'emplacement dans le tableau ou se situe la
valeur que l'on vient de renseigner.

Mais ici, le problème est un peu plus grave, puisque les cases ne sont
pas vidées lors de la suppression/modification d'une chaîne de
caractères, la chaîne de caractères remplaçante est insérée à la fin
du tableau, et la référence du champs que l'on vient de modifier est
mise à jour avec cette nouvelle position.

Il est difficile de savoir si cette case n'était référencée que par le
champs que l'on vient de modifier. Si plusieurs personnes s'appellent
DUPONT, et qu'on en modifie une pour mettre DUPOND, on ne peut pas
supprimer DUPONT puisque c'est utilisé par d'autres.

Conserver les textes *supprimés* (à un endroit) est donc nécessaire.
Mais c'est aussi problématique. Dans le cas ou ce texte n'est effectivement
utilisé qu'à un endroit (par exemple les notes personnelles), on garde
une information devenue inutile. Pire, on pourrait conserver des données
*sensibles*, qui auraient du être supprimées.

## Le nettoyage de la base

Nous voyons donc, que tout cela est problématique. D'un point de vue
de la consommation de ressources (consommation de mémoire excessive),
mais aussi d'un point de vue de la confidentialité.

Nous allons donc nous attaquer à la purge de cette base. Côté algorithmie,
c'est très simple. On commence par identifier :
- les personnes à supprimer : ce sont les personnes qui n'ont aucune
  information, aucune famille, aucune relation.
- les familles à supprimer : idem, les familles sans information ni
  personne.
- les chaînes de caractère à supprimer : celles qui ne sont
  référencées nulle part.

Pour se faire, nous allons parcourir les tableaux de données, et
marquer les données à conserver. Les éléments qui ne sont pas marqués
sont donc ceux à supprimer.

Ensuite, on crée un nouvelle base, avec des tableaux de longueur
`taille initiale - données à supprimer`, et on les remplis avec les
données à garder en prenant soin de mettre à jour à jour les
références.

Par exemple, on a marqué les chaines `12`, `42` et `56` pour suppression.
Si un individu pointe vers la chaîne `43` pour son champ `nom`, on
sait que le décalage est de `2` case, car `12` et `42` on été supprimés,
le nouvel index est donc `41`.

## Et alors, ça donne quoi ?

J'ai fais quelques essais sur des bases collaboratives de Geneanet:

- genea50com :
  140 personnes, 29785 familles et 69666 chaînes supprimées.
  genea50com.gwb/base : 206M -> 194M (-6%)
- pierfit :
  pierfit.gwb/base : 400M -> 353M (-11%)
- sanchiz :
  180 personnes, 7541 familles et 48865 chaînes supprimées.
  sanchiz.gwb/base : 155M -> 151M (-3%)
- wikifrat :
  4 personnes, 846 familles et 47038 chaînes supprimées.
  wikifrat.gwb/base : 234M -> 104M (-51%)

L'index strings.inx se retrouve aussi du coup bien allegé par ce passage.

## Et ça ne casse rien ?

Le résultat de gwu (export GW) est strictement identique avant et après
passage du ramasse miettes. Donc non.
