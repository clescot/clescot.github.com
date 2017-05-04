---
layout: post
title: "git pre-push : un hook pour empêcher la propagation des bugs"
date: 2014-01-18 16:05
comments: true
categories: [git, productivité,hook, push, intégration continue, TDD, tests]
---

# Etat des lieux

Les applications développées comportent inévitablement des bugs.
L'erreur étant humaine, cet état de fait ne surprend plus grand monde.

Pour parer à cette situation, sont développés en parallèle toutes sortes de tests, dont les *tests unitaires*. Nécessaires mais non suffisants, ceux-ci sont généralement les plus rapides à s'exécuter, permettant d'avoir un **feeback rapide**.
Leur intérêt n'est plus à démontrer.


Il est navrant en 2014, que des bugs détectés en amont par des tests se propagent.

# Une vérification par une chaîne d'intégration continue centralisée ?

Le premier réflexe pour régler ce problème, est d'utiliser les outils déjà en place. Un des outils qui s'est généralisé, est une chaîne d'intégration continue centralisée, type [jenkins](http://jenkins-ci.org). Cela permet de détecter un bug quand l'exécution des tests unitaires du build échoue.

Cette solution permet d'éviter de propager des bugs en production, l'équipe ne livrant que si le traitement sur la chaîne d'intégration continue s'exécute avec succès.

Néanmoins, **la vérification par la chaîne d'intégration centralisée ne prévient pas la propagation du bug aux autres membres de l'équipe qui travaillent sur la même branche"**, car le code a été poussé sur le repository distant. Les autres développeurs et jenkins se synchronisent ensuite sur ce même repository déjà buggé.

Le *feedack* vers l'équipe est donc **trop tardif** !
Essayons de trouver une autre approche, permettant d'avoir un feedback beaucoup plus court.

# Une vérification par une chaîne d'intégration continue décentralisée !

Le but de cette approche, est de contrôler **au plus tôt** le code, avant de le propager au sein de l'équipe. Ce contrôle doit s'effectuer avant tout partage.

## Un hook sur push


J'utilise depuis quelques semaines, une solution chez un client permettant d'exécuter les tests unitaires via Maven (`mvn clean test`) lors de la commande `git push`.

Depuis la version *1.8.2* de git  apparue en mars 2013, le déclenchement d'un hook est maintenant possible sur un push.
Un Hook (crochet en français ?), est un moyen d'exécuter un script personnalisé à certains moments (commit, push, merge, etc..). Il s'exécute avant l'action en question, et conditionne l'exécution de celle-ci.

Au passage, vous pouvez connaître votre version de git en exécutant la commande `git --version`.

Les hooks sont situés dans le sous-répertoire `hooks` du répertoire caché `.git` de votre repository. Des exemples de hook, finissant par `sample` sont déjà présents.


J'utilise le hook défini par [itty bitty labs](http://blog.ittybittyapps.com/blog/2013/09/03/git-pre-push/), en exécutant les tests unitaires via maven  : 

```bash
#!/bin/bash 

CMD="mvn clean test" # Command that runs your tests

# Check if we actually have commits to push
commits=`git log @{u}..`
if [ -z "$commits" ]; then
    exit 0
fi

current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

$CMD
RESULT=$?
if [ $RESULT -ne 0 ]; then 
       echo "failed $CMD"
       exit 1
fi

exit 0
```

Placez ce script intitulé `pre-push` dans le répertoire `hooks`.

Ayant des tests d'intégration qui peuvent être longs, seuls les tests unitaires sont exécutés à chaque push.
Ainsi, {"le Hook sur push permet d'éviter la propagation des bugs vers les autres membres de l'équipe, mais bloque le développement lors de son exécution"}.

Pour résumer, cette solution : 

* ne s'enclenche que sur un git push
* est bloquante

Elle s'adapte bien à mon usage, car il peut m'arriver d'effectuer des commit sur une branche, en cassant des tests unitaires de façon transitoire lors de refactorings. Néanmoins, il existe des situations où il est préférable de désactiver un hook.

Git permet cela sur [tous les hooks git](http://git-scm.com/book/fr/Personnalisation-de-Git-Crochets-Git) via l'option `no-verify` : 

```bash
git push origin mabranche --no-verify
```



## Un hook sur commit avec un repository local caché


[David Gageot](http://www.leszindeps.fr/p/David/Gageot/8aa5af7b3ad8ac1d013ad9ae27a60000), proposait déjà cette approche de [chaîne d'intégration continue décentralisée](http://blog.javabien.net/2009/12/01/serverless-ci-with-git/) sur son blog en 2009, mais via une autre solution. 

Sa solution est à base de hook git sur un commit. Ce hook clone le repository local dans un répertoire caché , et exécute les tests avant de valider le commit. En cas de succès, il propage le code sur le repository local initial. 

**Le hook sur commit permet une détection des bugs très tôt, et ne perturbe pas la fluidité du développement**.

Pour résumer, cette solution : 

* est basée sur un hook déclenché sur un commit
* est non-bloquante

Ainsi, elle permet de ne pas attendre le résultat des tests pour continuer à développer.

La solution de David Gageot, permet un contrôle **plus tôt** par rapport au hook sur git push, avec pour contrepartie la nécessité d'effectuer des commits plus rigoureux.

Il ne faut donc pas sortir des clous lors de chaque commit, quitte à exceptionellement désactiver le hook via l'option `no-verify` citée plus haut (option valide sur tous les hooks).

Effectivement, les commits sous git étant fréquents, il pourrait être désagréable d'être bloqué en attendant le résultat de tests unitaires de nombreuses fois par jour. La solution du clone du repository local dans un répertoire caché est une solution élégante, car elle permet de ne pas être bloqué dans ses développements.


# Conclusion

Si vous êtes très rigoureux, utilisez donc la solution proposée par David Gageot. Si vous n'êtes pas en situation de le faire sur votre projet, peut-être que l'approche *hook sur git push* répondra à votre besoin.
J'espère qu'un de ces hooks vous permettra d'éviter la propagation de bugs vers vos collègues.





