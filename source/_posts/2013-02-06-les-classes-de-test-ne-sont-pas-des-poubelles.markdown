---
layout: post
title: "Les classes de test ne sont pas des poubelles ! "
date: 2013-02-06 01:03
comments: true
categories: [junit,java,tdd,organisation,test]
published: true
---
{% img right /images/sign-34163_150.png 149 150 poubelle  %}
## ... ou comment structurer ses tests unitaires.

#Résumé
Je suis souvent confronté à des classes de test d'une longueur **gigantesque** dans les projets où j'interviens.
Leur contenu en un seul bloc est souvent illisible, quelquefois désorganisé et difficilement maintenable.

J'aborde dans cet article, différentes solutions d'organisation de ses classes de tests en les comparant.

##La solution proposée

Je propose d'organiser les tests via des **classes statiques internes** couplées à de l'héritage. Vous pourrez regrouper
 vos méthodes de test par méthode testée, les exécuter sélectivement et rendre lisible la classe de test.

#Constat

Les classes de test sont, quand les tests sont nombreux, très longues.
Il est difficile de les appréhender d'un seul coup d'oeil.

Pour tester une méthode avec [Junit](http://www.junit.org), on crée une nouvelle méthode
dans la classe de test pour chaque nouveau cas d'usage.

Ainsi, les nombreux cas de test impliquent de créer **beaucoup** de méthodes de test.


##Conséquence

Il est difficile de :

*    retrouver tous les cas de test d'une méthode
*    identifier les cas de test récurrents
*    lancer uniquement tous les cas de test d'une méthode
*    avoir une vue d'ensemble de la classe de test



##Exemple
Voici la classe `Foo.java` à tester :

{% codeblock lang:java %}

    public class Foo {

        public int add(Integer a,Integer b){
        	return a.intValue()+b.intValue();
    	}

    	public int substract(Integer a,Integer b){
        	return a.intValue()-b.intValue();
    	}
    }
{% endcodeblock %}

Cette classe ne comporte que deux méthodes, prenant chacune deux paramètres.

Malgré sa simplicité, celle-ci comprend pour chaque méthode de nombreux cas à tester :

*    les paramètres sont null
*    un des paramètres est null
*    les paramètres sont négatifs
*    les paramètres sont positifs
*    les paramètres sont pour le premier négatif, pour l'autre positif
*    les paramètres sont pour le premier positif, pour l'autre négatif
*    le résultat est négatif
*    le résultat est positif
*    le résultat est égal à zéro
*    le résultat dépasse la limite inférieure du type *Integer*
*    le résultat dépasse la limite supérieure du type *Integer*
*    etc...

En résulte ainsi, un code **illisible** (je vous épargne le code complet) :

```java

    import org.junit.Test;
    import static org.hamcrest.CoreMatchers.is;
    import static org.junit.Assert.assertThat;

    public class FooTest {
        private Foo foo = new Foo();

        @Test(expected = NullPointerException.class)
        public void testAdd_withNullArguments() throws Exception {
            foo.add(null, null);
        }

        @Test
        public void testAdd_withPositiveArguments() throws Exception {
            int result = foo.add(3, 4);
            assertThat(result,is(7));
        }

        ......
        ......
        ......

        @Test(expected = NullPointerException.class)
        public void testSubstract_withNullArguments() throws Exception {
            foo.substract(null, null);
        }

        @Test
        public void testSubstract_withPositiveArguments() throws Exception {
            int result = foo.substract(3, 4);
            assertThat(result,is(-1));
        }
        ......
        ......
        ......
    }
```

## Diviser (le code) pour mieux régner

Face à une classe qui grossit à outrance, il est nécessaire de répartir les cas de tests, par exemple suivant la méthode testée.

Essayons de lister les possibilités pour regrouper les cas de test d'une même méthode.

### Des débuts d'organisation : les commentaires
Pour mieux organiser ses tests, beaucoup insèrent des commentaires comme repères.

```java


    ////////////// début des tests de la méthode 'add'
    .......
    .......
    ///////////// fin des tests de la méthode 'add'
    ////////////// début des tests de la méthode 'substract'
    .......
    .......
    ///////////// fin des tests de la méthode 'substract'
    
```

#### Avantages
Les méthodes comprises entre les blocs de commentaires sont regroupées et associées à la méthode testée.

#### Inconvénients
Cette solution :

*    rajoute beaucoup de bruit dans la lisibilité de la classe
*    ne resiste pas au reformatage du code effectué avec certains éditeurs de texte
*    ne permet pas de ne choisir d'exécuter qu'un sous-ensemble des tests unitaires

Dans la même optique, [Robert Martin (Uncle Bob)](https://sites.google.com/site/unclebobconsultingllc/),
dans son [livre *Clean Code*](http://www.amazon.fr/gp/product/0132350882/ref=as_li_qf_sp_asin_tl?ie=UTF8&camp=1642&creative=6746&creativeASIN=0132350882&linkCode=as2&tag=clescotcom08-21),
 insiste sur l'importance de la lisibilité du code, et recommande d'éviter d'encombrer celui-ci avec ce genre de commentaires. Les commentaires ne permettent pas une bonne orgnaisation du code.

### Pas de régions *à la .NET* en Java

{% pullquote %}

A noter que du côté .NET, les directives de régions ont été ajoutées à destination des éditeurs, pour qu'ils puissent
cacher ou afficher des parties de code. Les régions sont présentes dans le code source, mais pas dans le MSIL (équivalent du *bytecode* Java).
Cette fonctionnalité a donc uniquement pour but d'améliorer la lisibilité, mais ne permet pas de raffiner l'exécution
des tests. {"Les régions .NET ne résolvent pas les problèmes d'organisation, et ne sont pas disponible du côté java."}

{% endpullquote %}

### Une fausse bonne idée : les catégories Junit

[Depuis junit 4.8](http://kentbeck.github.com/junit/doc/ReleaseNotes4.8.html), il est possible d'ajouter des annotations,
afin de regrouper des méthodes (ou des classes) de test en catégories (des sortes de tags).
Plusieurs catégories peuvent être ajoutées à une même méthode ou classe.

```java

    public class FooTestWithCategories {
        private Foo foo = new Foo();

        @Category(Add.class)
        @Test(expected = NullPointerException.class)
        public void testAdd_withNullArguments() throws Exception {
            foo.add(null, null);
        }

        @Category(Add.class)
        @Test
        public void testAdd_withPositiveArguments() throws Exception {
            int result = foo.add(3, 4);
            assertThat(result,is(7));
        }

        @Category(Substract.class)
        @Test(expected = NullPointerException.class)
        public void testSubstract_withNullArguments() throws Exception {
            foo.substract(null, null);
        }

        @Category(Substract.class)
        @Test
        public void testSubstract_withPositiveArguments() throws Exception {
            int result = foo.substract(3, 4);
            assertThat(result,is(-1));
        }
    }
    
```

#### Avantages

Les catégories Junit :

*	rattachent les tests aux méthodes testées.
*	permettent de lancer l'exécution de tous les tests d'une méthode particulière, via un Runner Junit nommé `Categories`, et une sélection des tests via `@Categories.IncludeCategory`.

```java

    @RunWith(Categories.class)
    @Categories.IncludeCategory(Add.class)
    @Suite.SuiteClasses( {FooTestWithCategories.class })
    public class MyTestSuite {

    }
    
```

#### Inconvénients

Les catégories Junit :

*	ne permettent pas d'isoler les tests d'une méthode par rapport aux autres :
	les tests peuvent avoir les bonnes catégories, mais être éparpillés au sein d'une grande classe de test...
*	elles nécessitent un Runner Junit particulier *Categories*.Elles ne permettent pas d'utiliser d'autres
 	Runner Junit (un seul Runner est spécifié par exécution).
*	l’isolation des cas de test récurrents n'est pas encouragé par cette option.

Les catégories Junit sont plutôt à utiliser pour différencier l'exécution de tests en fonction soit
de leurs dépendances techniques externes (tests d'intégration),de leur rapidité, ou d'un autre critère **non fonctionnel**.

### La solution proposée : utilisation des classes internes

[Junit 4.5](http://www.junit.org/node/401) apporte l'utilisation d'un Runner permettant de lancer des tests présents
dans des **classes statiques internes**:
*Enclosed*, qui est présent dans un package *expérimental* (org.junit.experimental.runners.Enclosed).

```java

    @RunWith(Enclosed.class)
    public class FooTestWithEnclosed {
	private Foo foo = new Foo();

        public static class Add {
            private Foo foo = new Foo();

            @Test(expected = NullPointerException.class)
            public void testWithNullArguments() throws Exception {
                foo.add(null, null);
            }

            @Test
            public void testwithPositiveArguments() throws Exception {
                int result = foo.add(3, 4);
                assertThat(result,is(7));
            }
        }

        public static class Substract {
            private Foo foo = new Foo();pens
            @Test(expected = NullPointerException.class)
            public void testWithNullArguments() throws Exception {
                foo.substract(null, null);
            }

            @Test
            public void testwithPositiveArguments() throws Exception {
                int result = foo.substract(3, 4);
                assertThat(result,is(-1));
            }
        }
    }
    
```

#### Avantages

*	rassemble dans une structure toutes les méthodes de tests liées à une méthode testée
*	permet d'exécuter sélectivement les cas de test d'une méthode

#### Inconvénients

*	le runner Junit est présent dans un package au nom qui implique une existence peu pérenne (`experimental`)
*	le runner ne lance que des méthodes présentes dans des classes statiques internes. Cet inconvénient ne semble pas gênant, car il semble rarement souhaitable de mélanger des organisations 		différentes (classes statiques internes et présence directe des méthodes) au sein d'une même classe.


### Améliorer la solution : héritage couplé aux classes statiques internes

On vient de voir l'intérêt d'utiliser les **classes statiques internes** pour regrouper les cas de test d'une méthode.
Afin de clarifier les cas de test récurrents à implémenter pour toute nouvelle méthode, nous pouvons utiliser
des **interfaces internes** dont héritera toute nouvelle classe statique interne.

```java


    @RunWith(Enclosed.class)
    public class FooTestWithEnclosedAndInheritance {
        private Foo foo = new Foo();

        private interface MyTest {
            void testWithNullArguments();
            void testWithPositiveArguments();
        }

        public static class Add implements MyTest {
            private Foo foo = new Foo();

            @Test(expected = NullPointerException.class)
            public void testWithNullArguments() {
                foo.add(null, null);
            }

            @Test
            public void testWithPositiveArguments(){
                int result = foo.add(3, 4);
                assertThat(result,is(7));
            }
        }

        public static class Substract implements MyTest {
            private Foo foo = new Foo();
            @Test(expected = NullPointerException.class)
            public void testWithNullArguments(){
                foo.substract(null, null);
            }

            @Test
            public void testWithPositiveArguments(){
                int result = foo.substract(3, 4);
                assertThat(result,is(-1));
            }
        }
    }
    
```

#### Avantages

*	Utilise une classe statique interne pour regrouper des cas de test, permet d'utiliser l'héritage pour homogénéiser
	l'implémentation de cas de test récurrents. 
*	évite la duplication de code.

### Inconvénients

*	si vous utilisez une classe abstraite qui n'a pas de méthodes de test, il est possible que le Runner Junit vous lance une exception.
	Pour régler ce problème, il faut ajouter l'annotation *@Ignore* à la classe.

## Conclusion

Arrêtons d'ignorer les bonnes pratiques de développement dans les tests, avec pour pretexte que ce code n'ira
pas en production ! Le code de test n'est pas du code jetable, mais est intimement lié au code qui sera déployé.

**Les qualités du code de test et de production doivent être du même niveau.**

Utilisons les structures du langage disponibles comme les **classes statiques internes** pour répartir les cas de test, et **l'héritage** pour identifier
les cas de test récurrents.
La non-répétition du code, sa lisibilité, et sa structure cohérente, permettent d'écrire des tests plus robustes,
plus maintenables et compréhensibles !


## Références

* [Junit](http://www.junit.org)
* [Junit 4.5](http://www.junit.org/node/401)
* [Junit 4.8](http://kentbeck.github.com/junit/doc/ReleaseNotes4.8.html)
* [clean code](http://www.amazon.fr/gp/product/0132350882/ref=as_li_tf_tl?ie=UTF8&camp=1642&creative=6746&creativeASIN=0132350882&linkCode=as2&tag=clescotcom-21)


