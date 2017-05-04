---
layout: post
title: "Metrics, pour mesurer efficacement les performances : les bases"
date: 2013-10-11 03:44
comments: true
published: true
categories: [performance, mesure,métrique]
---

# présentation de Metrics

Je vous propose dans cette série de 4 articles, de vous présenter la librairie [Metrics](https://github.com/codahale/metrics),
initié par la société [Yammer](https://www.yammer.com/).
Celle-ci permet de fournir des métriques au niveau applicatif et JVM.

Ce premier article, présente les différents types de mesures disponibles dans Metrics, leur usage et leur installation.

# pourquoi mesurer ?

[A l'instar des grands du web](http://blog.octo.com/les-patterns-des-grands-du-web-lobsession-de-la-mesure/), il devient primordial de mesurer les choses avant d'agir.
Cela permet de matérialiser, identifier et comprendre le fonctionnement interne en production de nos applications. Cela permet aussi de lever des freins techniques ou fonctionnels, notamment en comprenant la valeur métier de certaines fonctionnalités, les dysfonctionnements techniques ou le manque de performances. Pour cela, rien de tel qu'une mesure fiable, numérique, pour comprendre, et prendre des décisions. 

La librairie Metrics, est une bonne solution pour répondre à ce besoin.

## installation de metrics

Pour installer Metrics, il faut rajouter à son fichier maven `pom.xml`, la section suivante :

```xml
<dependencies>
    <dependency>
        <groupId>com.codahale.metrics</groupId>
        <artifactId>metrics-core</artifactId>
        <version>3.0.1</version>
    </dependency>
</dependencies>

```
### montée de version, renommage de package

A noter qu'à partir de la version 3, Metrics a pour package de base `com.codahale.metrics`, et non `com.yammer.metrics` (cas des versions 2.x).

## Isolation des métriques par application

Toutes les métriques de Metrics, sont stockées dans une instance de MetricsRegistry. Pour que plusieurs applications au sein de la même JVM, aient des métriques dissociées, il faut que chacune d'elles créent leur propre instance (chaque application étant isolée car possédant son propre classloader). Ceci peut soit se faire via le framework d'injection de dépendances (Spring, Guice), soit via un servletListener pour des applications JEE, ou via un champ statique public final de chaque application, accessible depuis l'extérieur de la classe.

```java
public static final MetricRegistry metrics = new MetricRegistry();
```

## les métriques disponibles

Le registre installé, nous pouvons créer les métriques dont nous avons besoin.

Voici les différents types de métriques disponibles :

* la jauge
* le compteur
* la mesure
* l'histogramme
* la mesure
* le timer



Lors de cette création, nous pouvons les identifier de façon unique dans le registre par les éléments suivants :

* groupe : la valeur par défaut est le package de la classe
* type : nom de classe
* nom : décrit le but de la métrique
* scope (optionel) : permet de différentier des métriques de plusieurs instances d'une même classe



### la jauge

La jauge représente la valeur **instantanée** de ce qui est mesuré. Cette mesure simple, est à utiliser quand **nous ne maîtrisons pas directement son incrément et son décrément.**
C'est le cas quand la valeur représente un système externe, ou quand celle-ci est maîtrisée par une librairie tierce. 

Un exemple d'usage de la jauge est le nombre de sessions actives dans une application web.

```java
registry.register(name(SessionStore.class, "active-sessions"), new Gauge<Integer>() {
    @Override
    public Integer getValue() {
        return store.getActiveSessions();
    }
});
```
#### la jauge JMX
Cette jauge spécialisée permet d'intégrer **facilement** dans le registre Metrics, une valeur exposée via JMX. Metrics peut donc devenir l'unique référentiel de métriques applicatives, en récupérant des données tierces via JMX, afin de les exposer via d'autres canaux (JMX, Graphite, Ganglia etc...).


```java
registry.register(name(SessionStore.class, "cache-evictions"),
                 new JmxAttributeGauge("net.sf.ehcache:type=Cache,scope=sessions,name=eviction-count", "Value"));
```
#### la jauge ratio

Cette jauge permet d'intégrer dans Metrics, le résultat du **rapport entre deux variables**.
```java
public class CacheHitRatio extends RatioGauge {
    private final Meter hits;
    private final Timer calls;

    public CacheHitRatio(Meter hits, Timer calls) {
        this.hits = hits;
        this.calls = calls;
    }

    @Override
    public Ratio getValue() {
        return Ratio.of(hits.oneMinuteRate(),
                        calls.oneMinuteRate());
    }
}
```

#### la jauge cachée

Cette jauge permet de réutiliser pendant un temps paramétrable via un **cache**, un résultat coûteux à récupérer.

```java
registry.register(name(Cache.class, cache.getName(), "size"),
                  new CachedGauge<Long>(10, TimeUnit.MINUTES) {
                      @Override
                      protected Long loadValue() {
                          // assume this does something which takes a long time
                          return cache.getSize();
                      }
                  });
```

#### la jauge dérivée

Cette jauge permet de calculer un résultat **à partir d'une jauge déjà présente** dans le registre de Metrics.


```java
public class QueueManager {
    private final Queue queue;

    public QueueManager(MetricRegistry metrics, String name) {
        this.queue = new Queue();
        metrics.register(MetricRegistry.name(QueueManager.class, name, "size"),
                         new Gauge<Integer>() {
                             @Override
                             public Integer getValue() {
                                 return queue.size();
                             }
                         });
    }
}
```

### le compteur

représente une mesure maîtrisée par l'application, **qui s'incrémente et se décrémente**.
Un exemple de compteur pourrait être le nombre de tâches planifiées en cours d'exécution à un instant t.
Tous les compteurs ont une valeur initiale de 0.

```java
private final Counter pendingJobs = metrics.counter(name(QueueManager.class, "pending-jobs"));

public void addJob(Job job) {
    pendingJobs.inc();
    queue.offer(job);
}

public Job takeJob() {
    pendingJobs.dec();
    return queue.take();
}
```

### l'histogramme

L'histogramme représente **la distribution statistique de valeurs d'un ensemble de données**.
Cela nous permet d'avoir des informations comme la moyenne, le minimum, le maximum, la médiane, la déviation standard, les quartiles ou percentiles.
Afin d'éviter un plantage, lorsque le calcul porte sur un grand nombre de données, Metrics réalise un échantillonage statistique représentatif, via différents algorithmes, appelés **Reservoirs**. 

Les réservoirs fournis, hormis le dernier présenté, ont la particularité d'avoir **une occupation mémoire restreinte**.

#### Des échantillons uniformes

Il existe *l'algorithme de Vitter*, qui produit des échantillons uniformes. Celui-ci est intéressant pour faire une **analyse à long terme**, mais n'est pas adaptée pour savoir si la distribution des données porte un changement significatif récent.


```java
private final static Histogram histogram= METRIC_REGISTRY.register("myLongTermHistogram",new Histogram(new UniformReservoir()));
```

#### Des échantillons récents

Il existe aussi un algorithme mettant un fort poids statistique sur des données récentes. Celui-ci est à privilégier lorsque l'on veut avoir une **vision récente de la distribution des évenements**.

Cette algorithme est utilisé par défaut dans les Timers.


```java
private final static Histogram histogram= METRIC_REGISTRY.register("myshortTermHistogram",new Histogram(new ExponentiallyDecayingReservoir()));
```

#### Des échantillons à fenêtre glissante

cet algorithme permet d'étudier la **distribution des N dernières données**.

```java
private final static Histogram histogram= METRIC_REGISTRY.register("",new Histogram(new SlidingWindowReservoir(1024)));
```


#### Des échantillons temporels à fenêtre glissante

cet algorithme permet d'étudier **la distribution des N dernières secondes**.
Un bémol est à apporter concernant cet algorithme, qui contrairement aux autres, porte sur la totalité des données des N dernières secondes, ce qui peut représenter **un volume en mémoire important**. Cet algorithme est aussi le plus lent.


```java
private final static Histogram histogram= METRIC_REGISTRY.register("lastTenMinutes",new Histogram(new SlidingTimeWindowReservoir(10,TimeUnit.MINUTES)));
```


### la mesure

La mesure mesure **la fréquence** à laquelle surviennent les évenements sur les périodes suivantes : 

* la durée de vie complète de l'application
* les 15 dernières minutes
* les 5 dernières minutes
* la dernière minute

Le nombre de requêtes par seconde pendant les 5 dernières minutes est un bon exemple de mesures.


L'alimentation de cette mesure se fait de la façon suivante : 
```java
private final Meter requests = metrics.meter(name(RequestHandler.class, "requests"));

public void handleRequest(Request request, Response response) {
    requests.mark();
}
```

### le timer

Le timer représente **un histogramme des durées et une mesure de la fréquence d'apparition**.

Un résulat issu de l'usage du timer serait : 

 à 1500 requêtes par seconde, notre latence augmente de 25 à 357 ms.

```java
final Timer timer = registry.timer(name(Filter.class, "requests"));

final Timer.Context context = timer.time();
try {
    // handle request
} finally {
    context.stop();
}
```

# Conclusion

J'espère que ce premier article sur Metrics, vous permettra d'avoir **une vue d'ensemble des différentes métriques disponibles**.

Les trois autres articles de cette série porteront, dans l'ordre, sur [l'intégration de Metrics dans une application web JEE](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-jee/), [l'intégration de Metrics avec spring et guice](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-spring-et-guice/) et [l'intégration de Metrics avec JDBC, logback et jersey](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-JDBC-logback-et-jersey/).

## Références

- [Metrics](http://metrics.codahale.com/manual/core/)
- [application example liée à cette série d'articles](https://github.com/clescot/metrics-example)

 