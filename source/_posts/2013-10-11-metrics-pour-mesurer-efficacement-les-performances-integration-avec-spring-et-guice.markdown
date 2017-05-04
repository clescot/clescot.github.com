---
layout: post
title: "Metrics, pour mesurer efficacement les performances : intégration avec spring et guice"
date: 2013-10-11 07:08
comments: true
published: true
categories: [performance, mesure,métrique,spring, guice]
---

# Présentation de Metrics

Je vous propose dans cette série de 4 articles, de vous présenter la librairie [Metrics](https://github.com/codahale/metrics),
initié par la société [Yammer](https://www.yammer.com/).
Celle-ci permet de fournir des métriques au niveau applicatif et JVM.

Ce troisième article, présente l'intégration de Metrics avec les librairies d'injection de dépendances Spring et Guice.


# Metrics avec Spring et Guice

[Le deuxième article](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-jee/) de cette série consacrée à Metrics, présentait une intégration "legacy" de Metrics dans une application JEE.
 
Néanmoins, Il existe des librairies qui intègrent facilement Metrics à Spring ou Guice.
L'avantage de ces librairies, est qu'elles permettent de simplifier l'usage de Metrics, via des annotations.
Les applications utilisant maintenant principalement les containers d'injection de dépendances, votre configuration ressemblera plutôt à un des exemples présentés ci-après.


## Avec Spring

Une librairie tierce permet l'intégration facile de Metrics avec les applications utilisant Spring (cette librairie supporte, au moment où j'écris l'article, la version 3.x de Metrics, contrairement à la librairie intégrant Metrics et Guice).

Au choix, soit la configuration via un fichier XML, soit par des annotations permet d'intégrer les deux librairies. Cette configuration nous permettra d'annoter nos classes hébergeant les métriques.

Il faut rajouter la dépendance suivante à votre fichier maven `pom.xml` : 
```xml
<dependency>
    <groupId>com.ryantenney.metrics</groupId>
    <artifactId>metrics-spring</artifactId>
    <version>3.0.1</version>
</dependency>
```
### la configuration Spring via xml

La configuration en xml, de l'intégration des deux librairies se fait comme suit : 
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:metrics="http://www.ryantenney.com/schema/metrics"
       xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
            http://www.ryantenney.com/schema/metrics
            http://www.ryantenney.com/schema/metrics/metrics-3.0.xsd">

    <!-- registry and reporters should be defined in only one context xml file -->
    <metrics:metric-registry id="registry" name="springMetrics" />
    <metrics:reporter type="console" metric-registry="registry" period="1m" />

    <!-- annotation-driven must be included in all context files -->
    <metrics:annotation-driven metric-registry="registry" />

    <!-- beans -->

</beans>
```
Comme indiqué dans les commentaires du xml précédent ([issu de la documentation du module d'intégration](https://github.com/ryantenney/metrics-spring)), la déclaration du registre et des reporters doit être réalisée dans un seul fichier xml, tandis que la balise liée aux annotations doit être présente dans chaque fichier xml spring.

### la configuration Spring via des annotations

De façon alternative, l'intégration se réalise en javaConfig comme suit : 

```java

@Configuration
@EnableMetrics
public class SpringConfiguringClass extends MetricsConfigurerAdapter {

    @Override
    public MetricRegistry getMetricRegistry() {
        return SharedMetricRegistries.getOrCreate("springMetrics");
    }

    @Override
    public void configureReporters(MetricRegistry metricRegistry) {
        ConsoleReporter.forRegistry(registry)
            .outputTo(output)
            .build()
            .start(1, TimeUnit.MINUTES);
    }

} 	
```



Cette intégration permet de : 

* créer les métriques sur les méthodes des beans via les annotations `@Timed`, `@Metered`, `@ExceptionMetered`, and `@Counted`.
* créer des jauges sur les membres des beans spring annotés avec `@Gauge`
* injecter via spring les variables représentant les métriques via l'annotation `@InjectMetric`
* insérer, dans le registre spécifique des healthchecks, toute classe étendant la classe `HealthCheck`
* créer des reporters Metrics et les lier au cycle de vie de Spring

Une fois l'intégration de Metrics via Spring réalisée, il devient très aisé via des annotations de rajouter des métriques à notre application. 

Exemple de classe annotée : 

```java
public class MyClass {
                @Timed
                public void timedMethod(){
                }

                @Metered
                public void meteredMethod() {
				}

                @Counted
                public void countedMethod() {
                }
}

```


## Avec Guice

[l'application web exemple liée à cet article](https://github.com/clescot/metrics-example), montre l'intégration de Metrics avec une application utilisant le framework d'injection de dépendances Guice.

L'application utilise Metrics 3.0.1, via une librairie `metrics-guice` qui n'a pas encore de version stable (à l'heure où j'écris cette article) supportant Metrics 3.x. J'ai donc forké le repository de metrics-guice pour juste upgrader la dépendence de Metrics vers la version 3.0.1. [Un repository metrics-guice dépendant de Metrics 3.0.1 a été crée pour cette occasion](https://github.com/clescot/metrics-guice).

Classiquement, l'intégration de Guice dans une webapp se réalise via l'installation d'un servletListener étendant `GuiceServletContextListener`.

```java
package com.clescot.rest;

import com.codahale.metrics.ConsoleReporter;
import com.codahale.metrics.JmxReporter;
import com.codahale.metrics.MetricRegistry;
import com.codahale.metrics.health.HealthCheckRegistry;
import com.codahale.metrics.servlets.HealthCheckServlet;
import com.codahale.metrics.servlets.MetricsServlet;
import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.servlet.GuiceServletContextListener;
import com.palominolabs.metrics.guice.InstrumentationModule;

import javax.servlet.ServletContext;
import javax.servlet.ServletContextEvent;
import java.util.concurrent.TimeUnit;

public class GuiceContextListener extends GuiceServletContextListener {


    private ServletContext servletContext;

    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        servletContext = servletContextEvent.getServletContext();
        super.contextInitialized(servletContextEvent);
    }

    @Override
    protected Injector getInjector() {

        Injector injector = Guice.createInjector(new HealthCheckModule(), new RESTModule(), new InstrumentationModule(),
                new JDBCMetricsModule()
        );

        //register registries in ServletContext
        MetricRegistry metricRegistry = injector.getInstance(MetricRegistry.class);
        HealthCheckRegistry healthCheckRegistry = injector.getInstance(HealthCheckRegistry.class);
        servletContext.setAttribute(MetricsServlet.METRICS_REGISTRY, metricRegistry);
        servletContext.setAttribute(HealthCheckServlet.HEALTH_CHECK_REGISTRY, healthCheckRegistry);

        //configure reporters
        JmxReporter.forRegistry(metricRegistry).build().start();
        ConsoleReporter.forRegistry(metricRegistry).build().start(10, TimeUnit.SECONDS);
        return injector;
    }
}


```
Un HealthCheckModule a été crée :

```java
package com.clescot.rest;

import com.codahale.metrics.health.HealthCheck;
import com.google.inject.multibindings.Multibinder;
import com.google.inject.servlet.ServletModule;
import com.palominolabs.metrics.guice.servlet.AdminServletModule;

public class HealthCheckModule extends ServletModule{

    @Override
    protected void configureServlets() {
        install(new AdminServletModule());
        Multibinder<HealthCheck> healthChecksBinder = Multibinder.newSetBinder(binder(), HealthCheck.class);
        healthChecksBinder.addBinding().to(DatabaseHealthCheck.class);

    }

}

```
Les deux autres modules (cf l[es sources de l'application exemple](https://github.com/clescot/metrics-example)) correspondent à : 

- l'enregistrement des ressources REST dans Guice (`RESTModule`)
- la configuration de la base de données et son monitoring (`JDBCMetricsModule`)
- un module pour lier Metrics et Guice (`InstrumentationModule`). 


Vous noterez l'intégration d'un `JMXReporter`, et d'un `ConsoleReporter` dans cette configuration, qui envoie dans la sortie standard toutes les 10 secondes (à des fins de test), un descriptif des métriques.

Les ressources REST créées et les servlets Metrics disponibles seront référencées dans le fichier `web.xml` (ou de façon alternative via des annotations pour des containers plus récents).

```xml
<web-app>
    <display-name>Archetype Created Web Application</display-name>
    <listener>
        <listener-class>com.clescot.rest.GuiceContextListener</listener-class>
    </listener>

<filter>
    <filter-name>guiceFilter</filter-name>
    <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
</filter>
    <filter-mapping>
        <filter-name>guiceFilter</filter-name>
        <url-pattern>/rest/*</url-pattern>
    </filter-mapping>

    <servlet>
        <servlet-name>AdminServlet</servlet-name>
        <servlet-class>com.codahale.metrics.servlets.AdminServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet>
        <servlet-name>healthcheck</servlet-name>
        <servlet-class>com.codahale.metrics.servlets.HealthCheckServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet>
        <servlet-name>ping</servlet-name>
        <servlet-class>com.codahale.metrics.servlets.PingServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet>
        <servlet-name>metrics</servlet-name>
        <servlet-class>com.codahale.metrics.servlets.MetricsServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet>
        <servlet-name>threads</servlet-name>
        <servlet-class>com.codahale.metrics.servlets.ThreadDumpServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>AdminServlet</servlet-name>
        <url-pattern>/admin</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>healthcheck</servlet-name>
        <url-pattern>/admin/healthcheck</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>ping</servlet-name>
        <url-pattern>/admin/ping</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>metrics</servlet-name>
        <url-pattern>/admin/metrics</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>threads</servlet-name>
        <url-pattern>/admin/threads</url-pattern>
    </servlet-mapping>
</web-app>

```

Les exemples de code d'une application Guice ayant ces annotations sont présents dans [le dernier article](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-JDBC-logback-et-jersey/) (les annotations sont portées par des ressources Jersey).

## conclusion

L'intégration de la librairie Metrics avec Spring ou Guice se réalise sans réelles difficultés majeures. [Le dernier article de cette série, porte sur l'intégration de librairies tierces, comme Jersey, les drivers JDBC ou logback](/blog/2013/10/11/metrics-pour-mesurer-efficacement-les-performances-integration-avec-JDBC-logback-et-jersey/).


## Références

* [Metrics](http://metrics.codahale.com/manual/core/)
* [Metrics-Spring](http://ryantenney.github.io/metrics-spring/)
* [Spring](http://spring.io/)
* [Guice](http://code.google.com/p/google-guice/)
* [un repository Metrics-guice dépendant de Metrics 3.0.1](https://github.com/clescot/metrics-guice)
- [application example liée à cette série d'articles](https://github.com/clescot/metrics-example)
