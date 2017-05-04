---
layout: post
title: "web-debug-tag : visualiser facilement les variables des jsp dans son navigateur"
date: 2013-05-03 00:15
comments: true
categories: [debug,jsp,variables,communication,JSON,web,java,tag]
---

# Un besoin de communication

Lors de mes missions, j'ai pu observer différentes organisations d'équipe de développement d'applications web. 

Dans certains cas, une sous-équipe est dédiée uniquement à la réalisation de la couche graphique (*le "front"*), et une autre à la partie serveur (*le "back"*). Ceci peut engendrer une "frontière" technique, une opacité quant aux variables disponibles dans les JSP fournies par les développeurs "back".


Afin d'améliorer **la communication entre les équipes**, [Jérémy Goupil](https://github.com/jeremyGoupil) et moi avons codé une taglib appelée [web-debug-tag](https://github.com/figarocms/web-debug-tag), permettant de **rendre visible** aux développeurs front ces variables.

Je vous présente donc dans cet article, son principe, son installation, et son usage, d'abord dans une application web simple, puis dans un cas plus complexe.

# Web-debug-tag, kesako ?

Cette taglib **sérialise** dans la réponse du serveur, au format **JSON**, les variables et leurs valeurs, qu'elles soient dans les scopes *page*, *request*, *session* ou *application*. 

Les valeurs des variables peuvent être aussi bien des chaînes de caractères que des arbres d'objets, le tag utilisant la librairie [Jackson](https://github.com/FasterXML/jackson-core) pour sérialiser le tout. Seuls les champs publics des objets, ou ceux ayant des accesseurs publics seront sérialisés.

De plus, une fois ces objets java sérialisés en JSON, du code client met dans la console javascript ce résultat pour être facilement visualisé dans le navigateur.

## Installation

L'installation de cette librairie, pour les projets sous maven, nécessite l'ajout dans votre fichier `pom.xml` de la section suivante : 

	<dependency>
		<groupId>fr.figarocms</group>
		<artifactId>web-debug-tag</artifactId>
		<version>1.6</version>
	</dependency>

Cet ajout permettra à votre projet d'inclure dans votre fichier `war`, le jar de la taglib.

Une fois installé, vous pouvez maintenant inclure dans vos JSP la directive suivante:

	<%@ taglib prefix="debug" uri="https://github.com/figarocms/web-debug-tag"%>

suivi par le tag en lui-même:

	<debug:debugModel/>

SI votre application comporte de nombreuses pages avec des parties communes, vous utilisez certainement un moteur de template. Placez donc cette directive dans un template global pour qu'il s'applique à toutes les JSP de l'application.

## À utiliser uniquement pour le développement


Cette librairie exposant toutes les variables de l'application côté navigateur, celle-ci n'est pas activée par défaut ; {"l'activation du web-debug-tag en production est totalement déconseillée"}, principalement pour des raisons de sécurité.

Pour l'activer, il faut donc mettre dans la ligne de commande lançant votre serveur d'applications le paramètre suivant:

	-Ddebug.jsp=true

Un message d'avertissement s'affichera néanmoins à chaque démarrage pour vous rappeler que ce n'est qu'une librairie dédiée aux environnements de développement.


## Une application web d'exemple

J'ai codé une application basique afin d'illustrer l'usage du web-debug-tag.

Pour tester le résultat, vous pouvez cloner le repository github par la commande suivante : 

	git clone https://github.com/clescot/web-debug-tag-example

Dans le répertoire *web-debug-tag-example* nouvellement crée, vous pouvez lancer la commande maven suivante pour lancer un serveur jetty:

	mvn org.eclipse.jetty:jetty-maven-plugin:9.0.2.v20130417:run-exploded

## Rendu

Le tag génère du code, qui met dans l'objet javascript `console` la grappe d'objets JSON générée.

voici un exemple de rendu sous ce même jetty:

{% img right /images/capture_web-debug-tag.png 1855 1056  les variables jsp apparaissent dans l'onglet console de firebug  %}


On peut voir dans cette capture d'écran, la sérialisation d'éléments présents dans les différents scopes de [l'application web créée pour l'article](https://github.com/clescot/web-debug-tag-example), via la console [Firebug](https://getfirebug.com) sous Firefox. D'autres navigateurs tels Chrome permettent de visualiser aussi la sortie de l'objet console.

On pourra noter dans cette capture, la sérialisation d'objets dans les différents scopes, que les objets soient simples (chaînes de caractères, entiers), ou complexes (objets contenus dans d'autres objets). 

## Exclusion de certaines classes non sérialisables

Certaines classes particulières, ne sont pas sérialisables comme des proxies Spring, Sitemesh ou Hibernate, sauf à utiliser [des modules Jackson tierces](https://github.com/FasterXML/jackson-module-hibernate). Ceci se traduit par une [JsonMappingException](http://fasterxml.github.io/jackson-databind/javadoc/2.2.0/com/fasterxml/jackson/databind/JsonMappingException.html) levée par [Jackson](https://github.com/FasterXML/jackson-core), qui englobe des StackOverflowError, syndrôme de la boucle infinie. 

Il est donc possible, via l'utilisation d'un paramètre de contexte dans le fichier `web.xml`, de définir les classes à exclure de la sérialisation via [une expression régulière](http://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html). A noter que ces classes peuvent être présentes à différents niveaux de la grappe d'objets à sérialiser.

voici un exemple de configuration d'exclusion évitant des problèmes liés aux classes de Tomcat 7, Jetty 9, sitemesh et Spring:

	<context-param>
		<param-name>webdebug.excludes</param-name>
		<param-value>org.springframework.*,__spring.*,__sitemesh.*,org.apache.jasper.*,org.apache.catalina.*,org.eclipse.jetty.webapp.Context,org.eclipse.jetty.server.*,org.eclipse.jetty.servlet.*,org.eclipse.jetty.webapp.*</param-value>
	</context-param>


## Utilisation du web-debug-tag dans une application web plus complexe

l'équipe du framework Spring, fournit [une application web exemple](https://github.com/SpringSource/spring-petclinic) à laquelle j'ai ajouté, [dans un fork du repository github](https://github.com/clescot/spring-petclinic), le web-debug-tag, pour tester une situation plus complexe.

voici sa configuration dans le `web.xml`, créée après plusieurs essais permettant d'éviter des JsonMappingException :

	 <context-param>
        <param-name>webdebug.excludes</param-name>
        <param-value>org.springframework.web.context.support.ServletContextResourcePatternResolver,org.apache.naming.*,
            org.springframework.aop.*,net.sf.ehcache.DiskStorePathManager,org.springframework.context.support.*,__spring.*,__sitemesh.*,org.apache.jasper.*,org.apache.catalina.*,org.eclipse.jetty.webapp.Context,net.sf.ehcache.CacheManager,org.apache.commons.dbcp.BasicDataSource,org.springframework.beans.factory.support.*,org.eclipse.jetty.server.*,org.eclipse.jetty.servlet.*,org.eclipse.jetty.webapp.*</param-value>
    </context-param>

Cette configuration pourrait être affinée si nécessaire, afin de voir plus d'informations liées à Spring (les développeurs web en ont-ils besoin ?).

voici un exemple de rendu des variables via le web-debug-tag : 


{% img right /images/Capture-spring-pet-clinic.png 1615 1026  les variables jsp apparaissent dans l'onglet console de firebug  %}

# conclusion

Je vous ai présenté l'installation et l'usage du web-debug-tag. Cet outil a eu un retour positif des développeurs "front" chez mon client.

J'espère que ce tag  vous sera utile, permettant une meilleure communication entre les parties "front" et "back", si vos équipes sont structurées ainsi.
