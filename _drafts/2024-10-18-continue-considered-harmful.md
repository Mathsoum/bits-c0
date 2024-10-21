---
layout: single
title: "`continue;` considered hardmful"
date: 2024-10-18 17:00:00 +0200
tags: c
---

Intro ?

## Cas d'usage rencontré

Le cas d'usage que j'ai rencontré et qui m'a donné envie d'écrire cet article est le
suivant. Les noms des variables et méthodes ont été changés pour assurer l'anonymat
des protagonistes. Cas simple, qu'on a sûrement déjà écrit des centaines (milliers)
de fois.

Une boucle itère sur une liste d'éléments, saute les éléments invalides, gèrent les cas
d'erreurs et nettoie après sont passage.

```c
zlist_t* list_elements; // Initialized & populated somewhere before
element_t *it = zlist_first(list_elements);
while (it) {
    if (!should_be_processed(it))
        continue;

    char* processed = process(it);
    if (!processed){
        log("There was a processing error");
        return;
    }

    store(processed);
    free(processed);
    it = zlist_next(list_elements);
}
zlist_destroy(&list_elements);
```

Du premier coup d'oeil, rien ne sort de l'ordinaire et on voit bien
les différents éléments évoqués dans l'énoncé.

"Une boucle itère sur une liste d'éléments"

```c
element_t *it = zlist_first(list_elements);
while (it) {
    // ...
    it = zlist_next(list_elements);
}
zlist_destroy(&list_elements);
```

"saute les éléments invalides"

```c
while (it) {
    if (!should_be_processed(it))
        continue;
    // ...
```

"gèrent les cas d'erreurs"

```c
char* processed = process(it);
if (!processed){
    log("There was a processing error");
    return;
}
```

"et nettoie après sont passage"

```c
    // ...
    free(processed);
    it = zlist_next(list_elements);
}
zlist_destroy(&list_elements);
```

## What could go wrong?

Vous en doutez bien, si je parle de ce bout de code, c'est que j'ai des
choses à dire dessus. En effet, plusieurs bug se cachent dans ce code.
Le code ayant été simplifié pour l'article, quelques minutes d'analyse
suffisent à les trouver.

Fun Fact : Il se trouve que j'en ai également trouver que je n'avais
pas vu en écrivant cet article. Il aura donc au moins servi à ça !

### `continue;` à l'infini

Le problème avec ce `continue;` est qu'il saute l'itération en fin de boucle.
La construction `while(it) { it = next(list); }` est une structure commune que
notre cerveau sait reconnaître sans effort.
Puis on ajoute une gestion des cas invalides en introduisant une construction
`if (invalid(it)) continue;` pour sauter ces cas-là et passer directement à
l'itération suivante.

Et c'est aussi rapidement que ça qu'on introduit une boucle infini. Dès lors
qu'on tombe dans le cas du `if (invalid(it))`, on exécute le `continue;` qui
relance un tour de boucle sans passer à l'itération suivante avec un
`it = next(list);`.

**Comment éviter ça ?**  
La version la plus simple : inverser les conditions pour éviter les `continue;`.
Bien que simple et rapide, il faut tout de même faire attention au fait que
l'avancement de l'itérateur doit être fait **en dehors** du bloc `if` qui
conditionne le traitement.

```c
while (it) {
    if (should_be_processed(it)){
        char* processed = process(it);
        if (!processed){
            log("There was a processing error");
            return;
        }

        store(processed);
        free(processed);
    }
    it = zlist_next(list_elements);
}
zlist_destroy(&list_elements);
```

Une autre solution est de convertir la boucle `while` en boucle `for`.

Les structures `while() {}` et `do {} while();` sont adaptées à des boucles
conditionnées fonction sur une variable d'état (e.g. isValid, found) ou
d'un cas d'erreur (e.g. retour de méthode, test sur erreur).
L'alternative lorsqu'on souhaite itérer sur une collection bornée est d'utiliser
les boucles `for (;;) {}`. L'avantage des boucles `for` dans notre exemple est
qu'il est impossible de "sauter" le code qui fait avancer à l'itération suivante.
Un autre avantage est que la variable utilisée pour l'itération peut être
généralement déclarée directemant dans la déclaration de la boucle `for` et
donc être restrainte au scope de la boucle.

Si on essaye de transformer la boucle `while` précédente en boucle `for`, on arrive au
résultat suivant :

```c
for (element_t *it = zlist_first(list_elements) ; it != NULL ; it = zlist_next(list_elements)){
    if (!should_be_processed(it))
        continue;

    // Processing...
    // No need to advance iterator here
}
zlist_destroy(&list_elements);
```

Pas besoin de gérer un cas particulier, utiliser `continue;` dans une boucle `for`
va forcément faire avancer l'itérateur.

Bien sûr, on peut aussi combiner les deux méthodes et utiliser un `for` sans avoir de `continue;`

```c
for (element_t *it = zlist_first(list_elements) ; it != NULL ; it = zlist_next(list_elements)){
    if (should_be_processed(it)){
        // Processing...
    }
}
zlist_destroy(&list_elements);
```

Bye bye la boucle infinie.

### Un nettoyage un peu rapide

Pour être totalement transparent, le code initial a déjà du code nettoyage dans
la gestion du cas d'erreur. Des variatbles locales déclarées avant la boucle
sont bien nettoyées avant de quitter la la méthode englobante avec le `return;`
en cas d'erreur. J'ai homis ce code dans les exemples pour ne pas détourner le
propos de cet article.

Le nettoyage en question dont on va parler dans ce paragraphe est le nettoyage
de la liste sur laquelle la boucle itère.
En effet, cette liste doit être détruite une fois la boucle terminée. C'est fait
via l'appel à zlist_destroy à la fin de la boucle.
Or, ce n'est pas fait dans la gestion du cas d'erreur. On a donc une fuite mémoire
en cas d'erreur vu on va quitter la méthode englobante et donc ne pas exécuter le
code de nettoyage de la liste.

La correction la plus rapide serait de simplement nettoyer la liste avant de faire le
return dans la gestion du cas d'erreur.

```c
zlist_t* list_elements; // Initialized & populated somewhere before
element_t *it = zlist_first(list_elements);
while (it) {
    if (!should_be_processed(it))
        continue;

    char* processed = process(it);
    if (!processed){
        log("There was a processing error");
        zlist_destroy(&list_elements);
        return;
    }

    store(processed);
    free(processed);
    it = zlist_next(list_elements);
}
zlist_destroy(&list_elements);
```

L'inconvenient de cette approche est qu'elle oblige à duppliquer le code de nettoyage
qui apprait dans plusieurs cas.
Si les cas d'erreurs se multiplie, on va se retrouver avec des bouts de code plus ou
moins similaires en charge de nettoyer les temporaires créées jusqu'à ce moment là.
Plusieurs parties suffisament similaires sont de bonnes cachettes pour les bugs.

La simplicité de l'exemple fait que remplacer le `return;` par un `break;` aura l'effet
escompté. En cas d'erreur, on sort directement de la boucle pour exécuter le code de
nettoyage juste après celle-ci.

Dans le cas d'une gestion d'erreur plus complexe, ou simplement si d'autres choses sont
faites après la boucle, une gestion d'erreur plus centrale peut rajouter plusieurs lignes
de code et même rendre la lecture plus difficile.

Sans en arriver à un `goto` vers un label qui s'occupe du nettoyage, on peut essayer
d'introduire une variable pour contrôler le succès de la boucle.

```c
bool processing_error_occured = false;
for (element_t *it = zlist_first(list_elements) ; it != NULL ; it = zlist_next(list_elements)){
    if (!should_be_processed(it))
        continue;

    char* processed = process(it);
    if (!processed){
        log("There was a processing error");
        processing_error_occured = true;
        break;
    }

    store(processed);
    free(processed);
}
zlist_destroy(&list_elements);
if (processing_error_occured){
    // Some cleanup
    return;
}
// The rest of the method...
// Same cleanup than before if there was no errors
```

En prenant un peu de recul, on voit que cette solutoin n'est pas idéale parce qu'elle
introduit un `return;` au milieu de la méthode englobante. C'est justement ce qu'on
essaye d'éviter avec notre boucle.  
Dans ce cas-là, inverser la condition d'erreur permet de continuer avec le reste du
traitement de la méthode et d'avoir le nécessaire de nettoyage à la fin de la méthode.

```c
zlist_t* list_elements; // Initialized & populated somewhere before
bool processing_error_occured = false;
for (element_t *it = zlist_first(list_elements) ; it != NULL ; it = zlist_next(list_elements)){
    if (!should_be_processed(it))
        continue;

    char* processed = process(it);
    if (!processed){
        log("There was a processing error");
        processing_error_occured = true;
        break;
    }

    store(processed);
    free(processed);
}
zlist_destroy(&list_elements);
if (!processing_error_occured){
    // The rest of the method...
}
// Locals cleanup
return;
```

Je trouve personnellement le if (error) return; plus clair. Cependant, selon
la taille de la méthode, il se peut que le return; soit perdu au milieu de
dizaines de lignes de code et donc être "oublié" lors des futurs développements.

C'est dans les endroits du code qui ne sont pas évidents que les bugs prolifèrent.

Un point notable sur ce dernier exemple : il est plus en accord avec les principes
de la programmation structurée. Chaque "bloc" peut être isolé en une fonction
séparée si besoin. Ce n'est pas le cas des boucles avec un return; en plein milieu.

_Je ne rentre pas dans les détails ici_
_Cette partie là dérive. Le sujet n'est pas la programmation structurée  
Par contre, le cas est sûrement valable pour une explication de la programmation structurée_

Le nettoyage dont je parle ici est le nettoyage de la liste sur laquelle la boucle
itère. Un return au milieu d'une boucle peut donc demander donc de considérer des
étapes supplémentaires concernant le nettoyage des variables locales. C'est une
charge cognitive supplémentaire qui peut être évitée en suivant les principes de la
programmation structurée.

## Conclusion

- Sur un exemple de code cours et simple, plusieurs choses sont à redire.
- Les corrections sont rapide et c'est considéré de la responsabilité des devs
  de corriger ce type de code quand on passe devant

## Ressources

_La liste des resources concerne la programmation structurée qui n'est finalement pas abordée dans cet article_

- [The Forgotten Art of Structured Programming - Kevlin Henney at Cpp On Sea 2019 (Youtube)](https://www.youtube.com/watch?v=SFv8Wm2HdNM)
- [Structured Programing - Wikipedia](https://en.wikipedia.org/wiki/Structured_programming)
