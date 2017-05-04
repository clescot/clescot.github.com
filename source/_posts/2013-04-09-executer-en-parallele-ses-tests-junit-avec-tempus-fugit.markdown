---
layout: post
title: "Exécuter en parallèle ses tests Junit avec tempus-fugit"
date: 2013-04-09 16:00
comments: true
categories: [tempus-fugit,junit,java,tdd,test,parallèle,multithread,testng,concurrence,thread-safe,testrule]
published: true
---

## ... ou comment tester si son code est **thread-safe**.

# Le problème

Lors du développement de services exposés sur internet (services web, servlets etc... ), il est primordial de s’assurer de la nature **thread-safe** du code.
	
Des problèmes d’écriture et lecture de données concurrentes, provoquant des erreurs difficilement reproductibles peuvent
survenir : un service sous forte charge pourrait se comporter de façon erronée sous forte charge. en bref, **le code est hors de contrôle**. 

# Une solution

L'approche habituelle pour détecter cette faiblesse est de favoriser la reproduction du problème par **l'exécution concurrente** (via plusieurs threads) et **répétée** du code.

## Concurrence et répétition via Tempus-fugit

Je vous propose d'exécuter en parallèle le code via des tests unitaires **[Junit]** et la librairie *[tempus-fugit]*.

Nous testerons pour cet article, la nature **thread-safe** d’un service REST développé avec Jersey. Ces tests mettront en oeuvre les annotations `@Concurrent`, `Repeating`, `@Intermittent` et le runner `ConcurrentTestRunner`.

Les exemples de code sont présent sur [un repository github dédié](https://github.com/clescot/tempus-fugit-example).

## Un exemple de service web


Voici un service web *[JAX-RS]*, qui présente deux points d'entrée permettant :

* d'ajouter un élément à une collection 
* d'effacer le contenu de la collection

Le code ci-dessous, a pour seul but d'illustrer les problèmes de concurrence d'accès. Ici, le problème vient de l'utilisation du champ de classe `coll`, sans synchronisation. L'accès à un champ de la classe, pour des services web, est la plupart du temps la partie critique du code à contrôler, que ce champ soit un champ d'instance ou un champ statique. Ainsi, il est nécessaire de garantir un accès unique au champ via du code synchronisé, ou de déclarer `volatile` le champ pour éviter que java ne cache sa valeur pour chaque thread. 
En bref, "utilisez le moins possible de champs d'instance pour des services web."


```java

	
	@Path("/test1")
	public class FooBar {
	
  	  private static Collection coll = Lists.newArrayList();
	
	
 	   @GET
 	   @Path("/path1")
  	  public Response test(@QueryParam("value") String value) throws InterruptedException {
  	      coll.add(value);
  	      int size = coll.size();
  	      return Response.ok(""+size).build();
	
 	   }
	
  	  @GET
  	  @Path("/clear")
  	  public Response test2(@QueryParam("value") String value){
 	       coll.clear();
 	       return Response.ok("ok").build();
  	  }
	}

```

# L'exécution parallèle

Comme indiqué précédemment, l'exécution parallèle est une des contraintes à imposer  au code pour favoriser l'émergeance du problème.

La librairie *tempus-fugit* nous propose deux mécanismes pour exécuter du code en parallèle :

* l'annotation `@Concurrent`
* le runner `ConcurrentTestRunner`

## L'annotation `@Concurrent`

La librairie *[Tempus-Fugit]*, fournit une *TestRule* , intitulée `ConcurrentRule`, qui s'active lorsque l'annotation `@Concurrent` est présente.



`"L'annotation @Concurrent permet d'exécuter une même méthode de test de façon concurrente sur plusieurs threads."`


A noter que les `TestRule` étaient anciennement appelées `MethodRule` jusqu'à [*Junit*] 4.8.

Les deux tests ci-dessous permettent de voir l'intérêt de cette *TestRule* :

* le premier test ne contient pas l'annotation `@Concurrent`. Il n'est donc exécuté qu'une seule fois, et s'exécute avec succès. Le non support d'une exécution concurrente **n'est donc pas détecté**.
* le second test est identique au premier, mais contient l'annotation `@Concurrent`. Ce Test est exécuté deux fois(`@concurrent(count=2)`, via deux threads distincts. Ce second test **permet de détecter** le non-support du code d'une exécution multi-threadée, notamment à cause de l'utilisation du champ de classe `coll`.


```java

	public class MultiThreadTest extends WebRunner {

	    private static Logger LOGGER = LoggerFactory.getLogger(MultiThreadTest.class);

  	    @Rule
	    public ConcurrentRule concurrentRule = new ConcurrentRule();

  
	    private WebResource webResource1 = resource().path("/test1/path1").queryParam("value", "4");

	    @Test
	    public void testResources_without_concurrent_annotation() {

	        for (long size = 1; size < 10; size++) {
	            assertThat(Long.parseLong(call(webResource1)), is(size));
	        }
 	       WebResource webResource2 = resource().path("/test1/clear");
	        call(webResource2);

	    }

	    @Test
	    @Concurrent(count = 2)
	    public void testResources_with_concurrent_annotation() {
	
	        for (long size = 1; size < 10; size++) {
	            assertThat(Long.parseLong(call(webResource1)), is(size));
	        }
  	      WebResource webResource2 = resource().path("/test1/clear");
	        call(webResource2);

	    }

	    private String call(final WebResource webResource) {

	        String response = null;
	        try {
  	          response = webResource.get(String.class);
    	        assertThat("path =" + webResource.getURI() + "response=" + response, !response.isEmpty(), is(true));
 	           LOGGER.info("response=" + response);
  	      } catch (UniformInterfaceException t) {
   	         Assert.fail(t.getMessage());
   	         LOGGER.error("erreur message=" + t.getMessage());
	            LOGGER.error("erreur réponse=" + t.getResponse());
	        }
	
	        return response;
	
	    }
	}

```

## Le runner `ConcurrentTestRunner`

A l'instar de l'annotation `@Concurrent`, un runner Junit permet d'exécuter l'ensemble des tests d'une classe chacun dans un Thread.
Ainsi, la classe de test suivante comporte 3 méthodes de test : 3 threads seront donc crées pour exécuter dans chacun d'eux une méthode de test en parallèle.

Le léger avantage de cette voie est la **simplicité** : il sufit de déclarer le runner pour que toutes les méthodes soient exécutés en parallèle ; pas besoin de déclarer la TestRule et d'annoter chaque test. 

Les deux principaux inconvénients sont le **manque de flexibilité** car on ne peut spécifier qu'un seul runner Junit pour une classe de test, et **le nombre de threads par méthode de test est fixe** (1 thread par test).

```java

@RunWith(ConcurrentTestRunner.class)
public class ConcurrentTest extends WebRunner{


        private static Logger LOGGER = LoggerFactory.getLogger(MultiThreadTest.class);

        private WebResource webResource1 = resource().path("/test1/path1").queryParam("value", "4");

        @Test
        public void first_test() {

            for (long size = 1; size < 10; size++) {
                assertThat(Long.parseLong(call(webResource1)), is(size));
            }
            WebResource webResource2 = resource().path("/test1/clear");
            call(webResource2);

        }

        @Test
        public void second_test() {

            for (long size = 1; size < 10; size++) {
                assertThat(Long.parseLong(call(webResource1)), is(size));
            }
            WebResource webResource2 = resource().path("/test1/clear");
            call(webResource2);

        }

        @Test
        public void third_test() {

            for (long size = 1; size < 10; size++) {
                assertThat(Long.parseLong(call(webResource1)), is(size));
            }
            WebResource webResource2 = resource().path("/test1/clear");
            call(webResource2);

        }

        private String call(final WebResource webResource) {

            String response = null;
            try {
                response = webResource.get(String.class);
                assertThat("path =" + webResource.getURI() + "response=" + response, !response.isEmpty(), is(true));
                LOGGER.info("response=" + response);
            } catch (UniformInterfaceException t) {
                Assert.fail(t.getMessage());
                LOGGER.error("erreur message=" + t.getMessage());
                LOGGER.error("erreur réponse=" + t.getResponse());
            }

            return response;

        }
}

```

# L'exécution répétée

La seconde contrainte à imposer au code est l'exécution répétée.

La librairie *tempus-fugit* nous propose deux mécanismes pour exécuter du code de façon répétée :

* l'annotation `@Repeating`
* L'annotation `@Intermittent` 

## L'annotation `@Repeating`

Souvent, la résolution des problèmes d'exécution concurrente de code n'est pas évidente, contrairement à l'exemple de cet article. Le bug se produit de façon erratique. Il est donc nécessaire de reproduire l'exécution des tests un grand nombre de fois, pour avoir la chance de faire échouer le code.

*Tempus-fugit*, nous fournit une autre *TestRule* intitulée `RepeatingRule`, qui est associée avec l'annotation `@Repeating`.

Elle permet d'exécuter un grand nombre de fois une méthode de test. Elle peut se cumuler avec l'annotation `@Concurrent`.

Voici un exemple d'une utilisation combinée de ces deux annotations :

```java

    @Rule
    public RepeatingRule repeatingRule = new RepeatingRule();

	@Test
	@Concurrent(count = 2)
	@Repeating(repetition=4)
	public void testResources_with_concurrent_annotation() {
	
	   for (long size = 1; size < 10; size++) {
	       assertThat(Long.parseLong(call(webResource1)), is(size));
	   }
  	   WebResource webResource2 = resource().path("/test1/clear");
	   call(webResource2);

	}
```

	
## L'annotation `@Intermittent`

Cette annotation a de grandes similitudes avec l'annotation `@Repeating`, à ceci prêt qu'elle n'est pas associée avec une `TestRule`, mais avec le *runner Junit* `IntermittentTestRunner`. Elle possède également un attribut nommé `repetition`.

Cette association a pour avantage d'exécuter **à chaque répétition**, les méthodes annotées par `@Before` et `@After`, contrairement aux `TestRule`.
{"Le principal inconvénient de @Intermittent est qu'on ne peut utiliser plus d'un runner Junit"}; on ne peut donc pas dans cette configuration utiliser un autre *runner*.

```java

@RunWith(IntermittentTestRunner.class)
public class IntermittentTest extends WebRunner{

    private static Logger LOGGER = LoggerFactory.getLogger(MultiThreadTest.class);
    private WebResource webResource1 = resource().path("/test1/path1").queryParam("value", "4");


    @Test
    @Intermittent(repetition = 30)
    public void test_method1_with_intermittent_annotation() {
        call(webResource1);
        LOGGER.info("method1 executed");
    }

    @Test
    @Intermittent(repetition = 10)
    public void test_method2_with_intermittent_annotation() {
        call(webResource1);
        LOGGER.info("method2 executed");
    }

    private String call(final WebResource webResource) {

        String response = null;
        try {
            response = webResource.get(String.class);
            assertThat("path =" + webResource.getURI() + "response=" + response, !response.isEmpty(), is(true));
            LOGGER.info("response=" + response);
        } catch (UniformInterfaceException t) {
            Assert.fail(t.getMessage());
            LOGGER.error("erreur message=" + t.getMessage());
            LOGGER.error("erreur réponse=" + t.getResponse());
        }

        return response;

    }
}

```

#Conclusion

La librairie *tempus-fugit* nous permet de tester la bonne exécution du code dans un environnement concurrent.
Elle nous offre deux alternatives pour mettre en place une exécution **parallèle** et **répétée** des tests unitaires Junit.
{"Tempus-fugit pallie au manque du support natif de Junit des contraintes de concurrence."} *TestNG*, l'autre librairie de test, intègre nativement ces problématiques via les attributs `threadPoolSize`, et `invocationCount`. A noter que *TestNG* inclut aussi, ~~contrairement à *Junit* et *tempus-fugit*~~, les attributs `timeout` et `invocationTimeOut` pour définir respectivement le temps que le test et l'ensemble des tests doivent prendre au maximum pour être considéré comme positifs.

*Tempus-fugit* fournit d'autres fonctionnalités, comme entre autres, une gestion facilitée des Threads (interruptions, pause, réveil), une gestion des verrous, et une détection des deadlocks.


## *Mise à jour du 13/04/2013*

Junit permet bien de définir un temps d'exécution maximum par méthode via le paramètre `timeout` de l'annotation `@Test`.

```java

//5 secondes maximum
@Test(timeout=500)
public void monTest() {
  ...
}

```



Junit permet aussi de définir un temps limite pour tous les tests d'une classe via la @Rule `TimeOut`.

```java

public class MaCLasseDeTest {

    @Rule
    public Timeout globalTimeout = new Timeout(8000); // 8 secondes maximum par méthode de test

    @Test
    public void monPremierTest() {
      ......
    }

    @Test
    public void monDeuxiemeTest() {
       ...   }
    }
}

```

## Références

* [Junit](http://www.junit.org)
* [tempus-fugit](http://tempusfugitlibrary.org)
* [JAX-RS](http://jax-rs-spec.java.net)
* [TestNG](http://testng.org)