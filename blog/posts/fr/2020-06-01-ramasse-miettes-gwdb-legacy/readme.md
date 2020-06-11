# Un ramasse-miette pour GeneWeb

Jusqu'ici, supprimer définitivement un individu sans passer par un
export/import gw n'était pas possible. C'est maintenant faisable, grâce à
l'outil `gwgc`.

## Préambule

GeneWeb, depuis peu, permet de paramétrer le moteur de stockage que
l'on souhaite utiliser pour enregistrer nos données. Cet article ne
concerne que le moteur de stockage historique (et au moment de
l'écriture le seul utilisable par GeneWeb). J'ai nommé `gwdb-legacy`.

## Un peu de technique

### Les chaînes de caractères (noms, prénoms, notes, ...)

Lorsque l'on enregistre le nom d'une personne, ce qui va etre enregistré dans la
personne, c'est en fait l'emplacement dans le tableau ou se situe la
valeur (chaîne de caractères) que l'on vient de renseigner.

Les chaînes de caractères sont stockées dans un tableau à part. Cela permet
de gagner espace de stockage car un même nom sera juste un même nombre, et la
vraie valeur ne sera stockée qu'une fois. Cela permet aussi une comparaison
beaucoup plus rapide, puisque ça revient à tester si deux nombres sont égaux, plutôt
qu'à comparer deux textes.

```
Positions             | 0  | 1   | 2        | 3      |        4 |      5 |
Chaînes de caractères | "" | "?" | "AZERTY" | "ZORG" | "DUPOND" | "DUPO" |
```

*Note : les valeurs 0 et 1 sont des valeurs spéciales, qui correspondent
toujours à la chaine vide et au point d'interrogation.*

Lors de la suppression/modification d'une chaîne de
caractères, la chaîne de caractères remplaçante est insérée à la fin
du tableau, et la référence du champs que l'on vient de modifier est
mise à jour avec cette nouvelle position.

Il est difficile de savoir si cette case n'était utilisée que par le
champs que l'on vient de modifier. Si plusieurs personnes s'appellent
DUPONT par exemple, et qu'on en modifie une pour mettre DUPOND,
on ne peut pas supprimer "DUPONT" du tableaux puisque c'est toujours
utilisé par d'autres personnes.

Dans le cas ou ce texte n'est effectivement utilisé qu'à un endroit
(par exemple pour les notes personnelles), on garde une information
devenue inutile.

Pire, on pourrait conserver des données *sensibles*, qui auraient du être supprimées.

### Les personnes et les familles

Les personnes et les familles sont stockées dans des tableaux également.

Chaque personne possède un identifiant numérique, c'est le numéro qui correspond à la
case mémoire ou sont enregistrés les informations de cette personnes. Ainsi, les
ressources sont liées entre elles (liens de parenté, témoins aux évènements, ...) en
utlisant ces identifiants numériques.

```
Positions  | 0                         | 1                         | 2                         |
Personnes  | { nom = 5 ; prenom = 10 } | { nom = 4 ; prenom = 12 } | { nom = 3 ; prenom = 12 } |
Traduction | { nom = "DUPO" ; ... }    | { nom = "DUPOND" ; ... }  | { nom = "ZORG" ; ... }    |
```

C'est pour cette raison, entre autres, qu'on ne peut pas simplement *décaler* le tableau
pour écraser une case à supprimer, car on modifierait alors les identifiant numériques de toutes
les cases suivantes.

Quand on supprime une personne, celle-ci n'est pas réellement
supprimée, mais la case qui contenait les données de la ressource est tout simplement vidée.

```
Positions  | 0                         | 1                            | 2                         |
Personnes  | { nom = 5 ; prenom = 10 } | { nom = 1 ; prenom = 1 }     | { nom = 3 ; prenom = 12 } |
Traduction | { nom = "DUPO" ; ... }    | { nom = "?" ; prenom = "?" } | { nom = "ZORG" ; ... }    |
```

On vide donc la case de toutes ses informations et on supprime les
liens vers/depuis cette case dans la base.

On crée en quelque sorte une nouvelle personne, vierge de toute
information et de tout référencement, pour l'insérer à la place de
ce qu'on supprime.

En résumé, on supprime les informations d'une personne, mais pas
l'objet `individu`.

Le problème, c'est que du coup cela créer des données fantômes,
les fameuse personness *? ?* que tout le monde cherche à supprimer de son arbre.
Il est possible de recycler les cases des
*fantômes* en éditant leur fiche pour re-remplir la case du tableau par
une vraie personne, au lieu de créer une nouvelle case, et c'est d'ailleurs
ce que font de nombreux utilisateur.

Mais ce n'est ni intuitif, ni pratique, et c'est actuellement
impossible de recycler les familles supprimées.

*Note: il serait enviseageable de recycler automatiquement les données
fantômes lors des nouvelles saisies, mais ce n'est pas le comportement
actuel de GeneWeb*

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

Pour se faire, nous allons parcourir les tableaux des personnes et
des familles données. Si la case contient des informations, on la marque
comme "à conserver", et on marque également les cases qu'elle références
(par exemple le nom, le prénom, les parents, ...) comme "à conserver"
également.

À la fin du parcours, les cases qui ne sont pas marquées sont donc celles à supprimer.

Ensuite, on crée un nouvelle base, avec des tableaux de longueur
`taille initiale - données à supprimer`, et on les remplis avec les
données à garder en prenant soin de mettre à jour à jour les
références.

Exemple :
```
                                       | 0  | 1   | 2        | 3      | 4        | 5      |
Tableau des noms avant ramasse-miettes | "" | "?" | "AZERTY" | "ZORG" | "DUPOND" | "DUPO" |
Tableau des noms après ramasse-miettes | "" | "?" | "DUPOND" | "DUPO" |

Personne avant ramasse-miettes { nom = 4 ; prenom = ... }
Personne après ramasse-miettes { nom = 2 ; prenom = ... }
```

## Et alors, ça donne quoi ?

Voici les résultat du ramasse-miettes sur quelques bases collaboratives de Geneanet :

- genea50com :
  - 140 personnes, 29785 familles et 69666 chaînes supprimées.
  - `genea50com.gwb/base` : 206M -> 194M (-6%)
- pierfit :
  - 3387 personnes, 22838 familles et 209406 chaînes supprimées.
  - `pierfit.gwb/base` : 400M -> 353M (-11%)
- sanchiz :
  - 180 personnes, 7541 familles et 48865 chaînes supprimées.
  - `sanchiz.gwb/base` : 155M -> 151M (-3%)
- wikifrat :
  - 4 personnes, 846 familles et 47038 chaînes supprimées.
  - `wikifrat.gwb/base` : 234M -> 104M (-51%)

*L'index `strings.inx` se retrouve aussi bien allegé par ce ramasse-miettes.*

## Et ça ne casse rien ?

Non. Pour s'en assurer, on peut comparer le résultat de gwu (export GW)
d'une même base avant/après passage du ramasse-miettes. Les deux fichiers seront
strictement identiques.
