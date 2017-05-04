---
layout: post
title: "Metrics, pour mesurer efficacement les performances : intégration avec JDBC, logback et jersey"
date: 2013-10-11 07:16
comments: true
published: true
categories: [performance,mesure,métrique,JDBC,logback,jersey]
---

# Présentation de Metrics

Je vous propose dans cette série de 4 articles, de vous présenter la librairie [Metrics](https://github.com/codahale/metrics),
initié par la société [Yammer](https://www.yammer.com/).
Celle-ci permet de fournir des métriques au niveau applicatif et JVM.

Ce quatrième article, présente l'intégration de Metrics avec les drivers JDBC, logback et jersey.



# avec les drivers JDBC

La librairie `JDBCMetrics` intégre JDBC avec Metrics. Cela permet : 

* d'avoir une vision globale de la charge de la base de données issue de votre application
* d'avoir une vision précise du nombre et des performances des requêtes SQL pour chaque requête HTTP

Pour rajouter ce module à votre application, il faut rajouter la dépendance suivante à votre fichier maven `pom.xml`: 
```xml
<dependency>
 <groupId>com.soulgalore</groupId>
 <artifactId>jdbcmetrics</artifactId>
 <version>1.1</version>
</dependency>

```

La vision globale de la charge de la base de données induite par l'application est possible via la configuration du driver JDBC, soit via un `Datasource` (la librairie jouant le rôle de proxy), soit via le `DriverManager`.

Au passage DriverManager est une classe dépréciée, ayant un comportement incohérent au niveau du chargement du driver. Préférez donc le Datasource.


La vision précise de la charge au niveau base de données par requête HTTP, est permise de façon optionnelle via l'installation d'un servlet filter : 
```xml
filter>
    <filter-name>JDBCMetricsFilter</filter-name>
    <filter-class>
        com.soulgalore.jdbcmetrics.filter.JDBCMetricsFilter
    </filter-class>
    <init-param>
        <param-name>use-headers</param-name>
        <param-value>true</param-value>
    </init-param>
    <init-param>
        <param-name>request-header-name</param-name>
        <param-value>jdbcmetrics</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>JDBCMetricsFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>

```


Pour avoir la mesure de la charge occasionnée par requête HTTP, il faut dans celle-ci mettre le header suivant :

`jdbcmetrics=yes`

La requête HTTP devra donc comporter cette entête supplémentaire, afin d'avoir une réponse incluant des entêtes spécifiques 
à JDBC.

voici un exemple de requête HTTP ayant cette entête (via l'extension Firefox RESTClient), ainsi que les entêtes de la réponse : 



{% img [center] /images/jdbc-metrics-custom-header.png requête HTTP pour jdbcmetrics %}

Les Métriques JDBC sont bien sûr visualisables via les reporters (ici via JMX) :

{% img [center] /images/metrics-jdbc.png métriques du driver JDBC %}

[L'application exemple de cette série d'articles](https://github.com/clescot/metrics-example), intègre Metrics et Guice, ainsi que JDBCMetrics.

Voici un exemple de module Guice, permettant d'installer le proxy JDBCMetrics entre le pool de connexions de la base et l'application : 

```java
package com.clescot.rest;

import com.google.inject.Binder;
import com.google.inject.Module;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class JDBCMetricsModule implements Module {

    public static final String JDBC_H2_URL = "jdbc:h2:mem:test";
    public static final String USERNAME = "sa";
    public static final String PASSWORD = "";
    public static final String CREATE_TABLE_FOR_AUDIT = "create table ACTIVITY (ID INTEGER auto_increment,STARTTIME datetime, ENDTIME datetime,  ACTIVITY_NAME VARCHAR(200),PRIMARY KEY (ID) )";

    @Override
    public void configure(Binder binder) {
        org.apache.tomcat.jdbc.pool.DataSource h2DataSource = new org.apache.tomcat.jdbc.pool.DataSource();
        h2DataSource.setUrl(JDBC_H2_URL);
        h2DataSource.setUsername(USERNAME);
        h2DataSource.setPassword(PASSWORD);
        h2DataSource.setDriverClassName(org.h2.Driver.class.getName());
        try(Connection connection = h2DataSource.getConnection()){
            PreparedStatement preparedStatement = connection.prepareStatement(CREATE_TABLE_FOR_AUDIT);
            preparedStatement.execute();

        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
        com.soulgalore.jdbcmetrics.DataSource metricsDataSourceProxy = new com.soulgalore.jdbcmetrics.DataSource(h2DataSource);
        binder.bind(DataSource.class).toInstance(metricsDataSourceProxy);
    }
}
```

# avec Logback

Metrics fournit une librairie d'intégration avec logback, pour remonter des informations concernant la fréquence des évenements logués **suivant le niveau de log**.

Pour intégrer Metrics et logback, il faut rajouter la dépendance suivante dans votre fichier `pom.xml` : 
```xml
<dependencies>
    <dependency>
        <groupId>com.codahale.metrics</groupId>
        <artifactId>metrics-logback</artifactId>
        <version>3.0.1</version>
    </dependency>
</dependencies>

```


Voici le code à utiliser pour lier Metrics à logback.

```java
final LoggerContext factory = (LoggerContext) LoggerFactory.getILoggerFactory();
final Logger root = factory.getLogger(Logger.ROOT_LOGGER_NAME);

final InstrumentedAppender metrics = new InstrumentedAppender(registry);
metrics.setContext(root.getLoggerContext());
metrics.start();
root.addAppender(metrics);
```

Une application concrète de cette intégration pourrait être une surveillance d'évenements logués en erreur ou warning, afin de réagir rapidement quand ceux-ci surviennent avec une fréquence importante.

Voici comment installer une mesure concernant les logs ayant le niveau `error` dans logback :


```java
registry.meter("ch.qos.logback.core.Appender.error");
```

A noter qu'une intégation avec log4J existe aussi.

# avec Jersey

Pour intégrer les mesures de Metrics avec les services REST exposés via Jersey et Spring, il est nécessaire d'intégrer le module suivant à votre fichier `pom.xml` : 
```xml
   <dependency>
         <groupId>com.yammer.metrics</groupId>
         <artifactId>metrics-jersey</artifactId>
         <version>3.0.1</version>
      </dependency>
```

## Sérialisation du registre Metrics en JSON via jackson


Le module maven `metrics-json`, permet de sérialiser facilement les mesures au format JSON, via des modules Jackson dédiés.

l'ajout de la dépendance suivante dans votre `pom.xml` permet de les utiliser : 

```java
<dependency>
    <groupId>com.codahale.metrics</groupId>
    <artifactId>metrics-json</artifactId>
    <version>3.0.1</version>
</dependency>
```


De plus, vous devez créer une ressource REST, qui va exposer la représentation JSON de votre registre Metrics comme dans l'exemple suivant : 

```java
@Path("/")
public class FooResource {
private static final ObjectMapper mapper = new ObjectMapper().registerModules(
            new com.codahale.metrics.json.MetricsModule(TimeUnit.SECONDS, TimeUnit.MILLISECONDS, false),new HealthCheckModule());
private MetricRegistry registry;

    @Inject
    public FooResource(MetricRegistry registry) {
        this.registry = registry;
    }

    @Path("/metrics")
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public String serializeMetricsRegistryInJSON() throws JsonProcessingException {
        return mapper.writeValueAsString(registry);
    }

}
```

Ainsi, cette ressource JAX-RS exposera sur l'url http://monhost:8080/metrics en GET une représentation JSON du registre Metrics.

Une exposition de ces métriques peut être utile, pour par exemple, une page de supervision habillant ces métriques avec du javascript.


## conclusion

La librairie Metrics est très pratique. Son usage s'est largement répandu, ce qui se traduit par la présence de librairies tierces afin d'enrichir son usage. Les 4 articles de cette série vous ont permis j'espère, de vous familiariser avec cette librairie. 
J'ai mis en place cette solution chez un de mes clients, en envoyant les informations de Metrics vers un serveur Graphite, pour une historisation pérenne, et un travail à postériori sur les métriques techniques ou fonctionnelles remontées.


Afin de distinguer les métriques des différents environnements (poste de développement, recette, pre-production, production...), remontées vers le même serveur, j'ai mis en place un `ServletContextListener` qui configure au démarrage de l'application le reporter Graphite en fonction de variables positionnées au lancement du serveur. Les métriques seront donc présentes dans graphite dans des arborescences séparées, via un préfixe différent.


```java
package com.clescot.listener;

import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import java.util.concurrent.TimeUnit;

import com.yammer.metrics.reporting.GraphiteReporter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MetricsGraphiteContextListener implements ServletContextListener {
    private static final String DEFAULT_GRAPHITE_HOST = "graphite";

    private static final String DEFAULT_GRAPHITE_PORT = "2003";

    private static final String DEFAULT_GRAPHITE_PREFIX = "default.graphite.prefix'";

    private static final String DEFAULT_GRAPHITE_PERIOD = "1";

    private static final Logger LOGGER = LoggerFactory.getLogger(MetricsGraphiteContextListener.class);

    private static final TimeUnit DEFAULT_GRAPHITE_TIME_UNIT = TimeUnit.MINUTES;

    private static final String GRAPHITE_HOST_SYSTEM_PROPERTY_KEY = "graphite.host";

    private static final String GRAPHITE_PERIOD_SYSTEM_PROPERTY_KEY = "graphite.period";

    private static final String GRAPHITE_TIME_UNIT_SYSTEM_PROPERTY_KEY = "graphite.time.unit";

    private static final String GRAPHITE_PORT_SYSTEM_PROPERTY_KEY = "graphite.port";

    private static final String GRAPHITE_PREFIX_SYSTEM_PROPERTY_KEY = "graphite.prefix";

    private static final String JAVA_SYSTEM_PARAMETER_PREFIX = " '-D";

    @Overridemetrics-example

    public void contextInitialized(final ServletContextEvent sce) {

        final String servletContextName = sce.getServletContext().getServletContextName();
        
        String graphiteHost = System.getProperty(GRAPHITE_HOST_SYSTEM_PROPERTY_KEY, DEFAULT_GRAPHITE_HOST);

        Long period = Long.parseLong(System.getProperty(GRAPHITE_PERIOD_SYSTEM_PROPERTY_KEY, DEFAULT_GRAPHITE_PERIOD));

        TimeUnit timeUnit = TimeUnit
                .valueOf(System.getProperty(GRAPHITE_TIME_UNIT_SYSTEM_PROPERTY_KEY, DEFAULT_GRAPHITE_TIME_UNIT.name()));

        int graphitePort = Integer
                .parseInt(System.getProperty(GRAPHITE_PORT_SYSTEM_PROPERTY_KEY, DEFAULT_GRAPHITE_PORT));

        String graphitePrefix = System.getProperty(GRAPHITE_PREFIX_SYSTEM_PROPERTY_KEY, DEFAULT_GRAPHITE_PREFIX);

        GraphiteReporter.enable(period, timeUnit, graphiteHost, graphitePort, graphitePrefix +"."+servletContextName);
        LOGGER.info(
                "graphite reporter enabled : period='{}', timeUnit='{}', graphite host='{}', graphite port='{}', metricsprefix='{}'",
                period, timeUnit, graphiteHost, graphitePort, graphitePrefix +"."+servletContextName);
        LOGGER.info("to customize graphite options listed above, put in the java command line some of these ones (without simple quotes):");
        LOGGER.info(JAVA_SYSTEM_PARAMETER_PREFIX + GRAPHITE_PERIOD_SYSTEM_PROPERTY_KEY + "=yourCustomGraphitePeriod'");
        LOGGER.info(
                JAVA_SYSTEM_PARAMETER_PREFIX + GRAPHITE_TIME_UNIT_SYSTEM_PROPERTY_KEY + "=yourCustomGraphiteTimeUnit'");
        LOGGER.info(JAVA_SYSTEM_PARAMETER_PREFIX + GRAPHITE_HOST_SYSTEM_PROPERTY_KEY + "=yourCustomGraphiteHost'");
        LOGGER.info(JAVA_SYSTEM_PARAMETER_PREFIX +GRAPHITE_PORT_SYSTEM_PROPERTY_KEY+"=yourCustomGraphitePort'");
        LOGGER.info(JAVA_SYSTEM_PARAMETER_PREFIX +GRAPHITE_PREFIX_SYSTEM_PROPERTY_KEY+"=yourCustomGraphitePrefix'");
    }

    @Override
    public void contextDestroyed(final ServletContextEvent sce) {

    }
}

```


## Références

- [Metrics](http://metrics.codahale.com/manual/core/)
- [Logback](http://metrics.codahale.com/manual/logback/)
- [Jersey](https://jersey.java.net)
- [JDBCMetrics](https://github.com/soulgalore/jdbcmetrics)
- [application example liée à cette série d'articles](https://github.com/clescot/metrics-example)


