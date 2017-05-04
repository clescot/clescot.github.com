---
layout: post
title: "Metrics pour mesurer efficacement les performances : intégration avec JEE"
date: 2013-10-11 05:01
comments: true
published: true
categories: [performance, mesure,métrique,JEE]
---

## Présentation de Metrics

Je vous propose dans cette série de 4 articles, de vous présenter la librairie [Metrics](https://github.com/codahale/metrics),
initié par la société [Yammer](https://www.yammer.com/).
Celle-ci permet de fournir des métriques au niveau applicatif et JVM.

Ce deuxième article, présente l'intégration de Metrics dans une application web JEE.
A noter que l'intégration proposée n'inclut pas, à des fins pédagogiques, une intégration facilitée via Spring ou Guice, comme c'est le cas dans [le troisième article de la série](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-spring-guice-logback-et-jersey/).


## Intégration dans une webapp

[Un exemple d'application web monitoré avec Metrics](https://github.com/clescot/metrics-example) est fourni en complément de cet article.



### Enregistrement du registre Metrics dans le scope application

Lors du démarrage de l'application JEE, il est nécessaire d'enregistrer dans le scope **application** (*i.e* le *servletContext*), le registre Metrics, pour qu'il soit disponible pour toutes les classes.
Nous utiliserons pour cette tâche, un *ContextListener*, qui est justement exécuté au démarrage de l'application.

Pour faciliter le partage du registre, Metrics fournit un module de classes utilitaires pour les applications JEE, intitulé `metrics-servlet`.

Installez-donc celui-ci dans votre fichier maven `pom.xml`.

```xml
      <dependency>
          <groupId>com.codahale.metrics</groupId>
          <artifactId>metrics-servlet</artifactId>
          <version>3.0.1</version>
      </dependency>
```


Une fois cette dépendance installée, vous pouvez étendre le `InstrumentedFilterContextListener` pour vous faciliter la tâche comme suit : 

```java
package com.clescot.rest;

import com.codahale.metrics.MetricRegistry;
import com.codahale.metrics.servlet.InstrumentedFilterContextListener;

public class MetricsListener extends InstrumentedFilterContextListener {

    private final static MetricRegistry METRIC_REGISTRY = new MetricRegistry();

    @Override
    protected MetricRegistry getMetricRegistry() {
        return METRIC_REGISTRY;
    }
}
```
L'installation de la classe MetricsListener effectuée dans votre fichier `web.xml` comme suit, vous pouvez utiliser les différentes métriques listées dans [le premier article de cette série](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-les-bases/) (jauge, compteur, mesure, histogramme, timer etc...).


```xml
...
...
 <listener>
        <listener-class>com.clescot.rest.MetricsListener</listener-class>
    </listener>
...
...
```

Veuillez noter que le module `metrics-servlet`, n'est pas à confondre avec `metrics-servlets` évoqué plus loin dans cet article.

Une alternative, présentée dans [le troisième article de cette série](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-spring-et-guice/), est l'initialisation et l'injection du registre via un container d'injection de dépendances comme [Spring](http://projects.spring.io/spring-framework/) ou [Guice](http://code.google.com/p/google-guice/) ([l'application web exemple]() utilise Guice). 

### Health check

Metrics permet aussi d'intégrer un système de "Health check", afin de surveiller, via des appels du répartiteur de charge (*load-balancer*), les composants externes sur lesquels repose l'application, tels la base de données, ou le moteur de recherche par exemple. 
Cela permet d'avoir une idée précise, de la fiabilité de ces composants, et donc de travailler sur la tolérance de l'application aux pannes de ceux-ci.

Cette intégration est réalisée par le module maven intitulé **metrics-healthchecks**.
Rajoutez donc dans votre fichier maven `pom.xml`, la dépendance suivante : 

```xml
<dependencies>
    <dependency>
        <groupId>com.codahale.metrics</groupId>
        <artifactId>metrics-healthchecks</artifactId>
        <version>3.0.1</version>
    </dependency>
</dependencies>

```

Puis il faut étendre la class HealthCheck pour en créer un :


```java
package com.clescot.rest;

import com.codahale.metrics.core.HealthCheck;

import javax.inject.Inject;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class DatabaseHealthCheck extends HealthCheck {
    private static final String DATABASE_HEALTH_CHECK_NAME = "database";

    private DataSource datasource;

    @Inject
    public DatabaseHealthCheck(DataSource datasource) {
        super("com.clescot.rest.DatabaseHealthCheck");
        this.datasource = datasource;
    }

    @Override
    protected Result check() throws SQLException {

        HealthCheck.Result result;
        try (Connection connection = datasource.getConnection()) {
            Statement statement;
            ResultSet resultSet;
            statement = connection.createStatement();
            resultSet = statement.executeQuery("select 1 from dual");
            if (resultSet.next()) {
                result = Result.healthy("'select 1 from dual' : OK");
            } else {
                result = Result.unhealthy("la requête 'select 1 from dual' retourne un résultat vide : KO");
            }
        } catch (Throwable t) {
            result = HealthCheck.Result.unhealthy("'select 1 from dual' : KO ", t);
        }

        return result;
    }

}


```

La création et l'enregistrement dans le registre de Metrics du healthcheck peut se faire dans la classe MetricsListener présentée précédemment. Le datasource pourra être récupérée si besoin, via JNDI. 

```java
registry.register("database", new DatabaseHealthCheck(datasource));
```

Enfin, il faut enregistrer dans le fichier `web.xml`, la servlet appelant les différents healthChecks enregistrés dans le registre, c'est-à-dire la healthCheckServlet :

```xml
...
	<servlet>
<servlet-name>healthcheck</servlet-name>
<servlet-class>com.codahale.metrics.servlets.HealthCheckServlet<</servlet-class>
<load-on-startup>1</load-on-startup>
</servlet>
...

<servlet-mapping>
<servlet-name>healthcheck</servlet-name>
<url-pattern>/healthcheck</url-pattern>
</servlet-mapping>
...

```
Le load balancer appelera donc l'url http://monapplication:8080/healthcheck à intervalles réguliers sur chaque instance de webapp pour s'assurer du bon fonctionnement de tous les noeuds du cluster. 

En cas de défaillance, l'instance en question sortira du pool d'instances utilisé pour répondre aux requêtes.

### Exposition des mesures
Une fois les mesures posées, il faut les exposer, pour pouvoir les visualiser, et donc les analyser. Ce sont les reporters  dans Metrics qui ont cette fonction d'exposition des métriques.

#### Via JMX
L'exposition des mesures Metrics, peut être réalisée via JMX (**non conseillé par Metrics en production**) par l'installation et l'initialisation d'un reporter spécifique de la façon suivante : 

```java
final JmxReporter reporter = JmxReporter.forRegistry(registry).build();
reporter.start();
```

Cette initialisation pourra aussi être effectuée dans le MetricsListener évoqué précédemment.

L'utilisation d'outils tels [jconsole](http://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html), ou [visualVM](http://visualvm.java.net/), vous permettra de visualiser les mesures exposées sous forme de MBeans.

Voici un exemple de visualisation des métriques JMX via visualVM :
{% img [center] /images/visualvm.png [métriques JMX via visualVM[métriques JMX via visualVM]] %}

#### Via HTTP

Metrics fournit par défaut, pour les applications JEE, des servlets permettant de sérialiser en HTTP le contenu du registre Metrics.

Pour obtenir ces servlets prêtes à l'emploi, il faut importer le module maven dans votre `pom.xml` :

```java
<dependency>
    <groupId>com.codahale.metrics</groupId>
    <artifactId>metrics-servlets</artifactId>
    <version>3.0.1</version>
</dependency>
```

Le module dédie expose les servlets suivantes : 

- `MetricsServlet `: expose les mesures via un objet JSON

Voici un exemple du rendu de cette servlet :
{% img [center] /images/metrics-http.png [métriques JSON[métriques JSON via MetricsServlet]] %}

- `HealthCheckServlet` : répond aux requêtes GET en exécutant tous les healthChecks enregistrés dans le registre. Cette classe est utile pour les loadBalancers, pour ne rediriger les requêtes des clients que vers les serveurs en bonne santé. Un code HTTP 200 est retourné en cas de succès, un code HTTP 500 est retourné dans le cas inverse.
- `ThreadDumpServlet` : renvoie une représentation textuelle de tous les threads en cours sur la JVM, c'est-à-dire leur état, leur stack trace, les verrous présents etc... Cette servlet est utile à des fins de diagnostic.
- `AdminServlet` agrège les services des servlets précédemment listées.

Les servlets listés sont certes très utiles pour exploiter une application ou aider au diagnostic d'un problème, mais elles ne sauraient être accessibles à tous. 

Il est donc nécessaire, soit via votre serveur web (apache, nginx etc..), soit via votre reverse proxy, ou votre firewall, d'empêcher un accès direct depuis l'extérieur. 

#### via d'autres canaux

d'autres reporters sont fournis par la librairie Metrics, pour exporter les métriques suivant différents canaux donc ceux-ci : 

- `ConsoleReporter` pour un export sur la sortie standard
-  `Slf4jReporter`pour un export via la façade de log `SLF4J`
- ` GraphiteReporter` pour stocker de façon scalable et pseudo temps-réel les métriques


# Conclusion

L'intégration de Metrics dans une application JEE est aisée. Nous verrons dans [le prochain article de la série, une intégration facilitée via Spring ou Guice](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-spring-et-guice/).

## Références

* [Metrics](http://metrics.codahale.com/manual/core/)
* [Metrics-Spring](http://ryantenney.github.io/metrics-spring/)
* [Spring](http://spring.io/)
* [Guice](http://code.google.com/p/google-guice/)
* [un repository Metrics-guice dépendant de Metrics 3.0.1](https://github.com/clescot/metrics-guice)
- [application example liée à cette série d'articles](https://github.com/clescot/metrics-example)

