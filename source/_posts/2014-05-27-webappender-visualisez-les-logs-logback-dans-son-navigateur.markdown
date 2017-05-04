---
layout: post
title: "webappender : pour visualiser les logs logback dans son navigateur"
date: 2014-05-27 23:46
comments: true
categories: [logback,web,logs,debug,communication]
---


Cette série de deux articles décrit le fonctionnement de la librairie de visualisation de logs [webappender](https://github.com/clescot/webappender). 

Le premier article expose sa mise en place, et les différentes façons de visualiser les logs suivant son navigateur.

Le deuxième article décrit la façon de choisir le contenu et la taille des logs à afficher.

# Le problème

Pour des applications web, il est courant de d'effectuer la recette des fonctionnalités sur des serveurs **distants**.

 Quand un comportement non attendu survient, il est nécessaire de retracer le fonctionnement précis de l'application, notamment via l'inspection des traces applicatives aussi appelées logs.

Le serveur étant distant, il est souvent nécessaire :

* de se connecter en ssh sur le bon serveur
* d'identifier le répertoire hébergeant les logs
* d'identifier le bon fichier de logs
* de démêler les logs correspondants à son test, des autres logs

Ces opérations ne sont pas instantanées et rébarbatives. 

# Webappender

Afin d'éviter toutes ses étapes, j'ai créer le webappender, afin que les traces serveur issues de vos requêtes HTTP arrivent directement dans votre navigateur.

Ainsi, les autres logs des requêtes exécutées de façon concurrente sur le serveur, ne sont pas envoyées vers votre navigateur. Le développeur regarde donc **uniquement** les informations qui sont liées à son test.


# Mise en place dans votre application web JEE

## Pré-requis

Webappender est compatible avec les applications [JEE](http://en.wikipedia.org/wiki/Java_Platform,_Enterprise_Edition) utilisant la librairie de logs [logback](http://logback.qos.ch).
L'adhérence entre webappender et JEE est minime. Une compatibilité plus large est donc possible.

## Installation en 2 étapes

### Maven

Ajouter à votre fichier Maven `pom.xml` cette dépendance :
   
```xml
	<dependency>
	  <groupId>com.clescot</groupId>
	    <artifactId>webappender</artifactId>
	    <version>1.4</version>
	</dependency>
```

Notez qu'un filtre de servlet, installé par annotation, est fourni avec la librairie. Son *urlPatterns* correspond à toutes les requêtes (`/*`).

### Activer le webappender
Par défaut, webappender est **désactivé** .

**Permettre de visualiser les logs de chacun (par une interception du réseau), peut être très risqué et dangereux dans des environnements de production**.

Donc, pour prévenir toute erreur de configuration, nous désactivons par défaut le webappender.

Pour l'activer, vous devez mettre ce paramètre sur la ligne de commande qui lance votre serveur d'applications :
	`-Dwebappender=true`.

# Visualisation des logs

La librairie webappender permet de visualiser les logs de sa requête dans tous les navigateurs.
Les visualisations de logs dans les configurations suivantes, impliquent une installation de webappender dans votre application JEE utilisant logback, avec un serveur d'applications java actif.

## Vos logs dans Chrome via ChromeLogger

Vous devez installer le plugin chrome intitulé [ChromeLogger](https://chrome.google.com/webstore/detail/chrome-logger/noaneddfkdjfnfdakjjmocngnfkfehhd).


Malheureusement, *Chrome logger* ne transmet par défaut aucun header l'identifiant, contrairement à d'autres plugins.

Or, Webappender transmet les logs à votre navigateur, quand les requêtes contiennent un header spécial : `X-ChromeLogger`.


Pour réparer cela, vous devez installer une extension comme [Modify Headers for Google Chrome](https://chrome.google.com/webstore/detail/modify-headers-for-google/innpjfdalfhpcoinfnehdnbkglpmogdi).

Veuillez noter que cette extension envoie à chaque requête, **que vous soyiez sur votre site web avec webappender ou non**, les entêtes spécifiques que vous avez rajouté. La publication des entêtes spécifiques vers les serveurs est donc très large.

Une alternative à *Modify Headers for Google Chrome*, est l'extension [ModHeader](https://chrome.google.com/webstore/detail/modheader/idgpnmonknjnojddfkpgkljpfnnfcklj), qui permet **par site** de définir les headers à modifier ou rajouter.




![header à ajouter pour activer chrome logger](/blog/images/modify_headers_chrome.png)


Quand l'installation et la configuration de cette nouvelle extension est effective, appuyez sur la touche `F12` de votre clavier, pour visualiser le panel *console* de Chrome. Cela vous montrera les logs engendrées par votre navigation sur la webapp. 

![logs affichées dans la console chrome logger](/blog/images/chrome_logger2.png)


Voici le détail de ce qu'une log contient : 

![détail d'une logs affichée dans la console chrome logger](/blog/images/capture_trace.png)


## Vos logs dans Firefox via FireLogger

Vous devez installer le plugin firefox intitulé [FireLogger](https://addons.mozilla.org/fr/firefox/addon/firelogger/).

Quand cela est fait, appuyez sur la touche `F12`, pour visualiser le panneau firebug ; vous devriez voir un nouvel onglet intitulé *logger*.
![logs affichées dans la console firelogger](/blog/images/capture_firelogger.png)

 Il vous montrera vos logs, suivant leur niveau. Les logs sont transmis à votre navigateur, quand les requêtes contiennnent une entête spéciale construite par le plugin 'FireLogger' : `X-FireLogger`.

![détail de log dans la console firelogger](/blog/images/capture_fire_logger_2.png)

## Vos logs dans n'importe quel navigateur

Si vous ne pouvez ou ne voulez pas installer un plugin firefox ou chrome, ou si vous n'avez aucun de ces navigateurs, vous pouvez tout de même visualiser vos logs dans n'importe quel navigateur. Dans ce cas, vous devez configurer un `body formatter`.

Pour cela, il faut :

- transmettre au serveur une entête spéciale : `X-BodyLogger`, via une extension comme `modify headers` (disponible sur chrome ou firefox), ou tout autre plugin compatible avec votre navigateur ayant la possibilité d'ajouter à façon une entête spéciale.


Pour Internet Explorer, une solution gratuite comme [Fiddler](http://www.telerik.com/fiddler) permet de modifier les entêtes HTTP.

![header HTTP activant un bodyformatter](/blog/images/body_logger.png)

- installer au début de vos  JSP, la declaration de la taglib, et à la fin de vos JSP, le tag webappender. Si vous utilisez une librairie de templating (comme `sitemesh` par  example), une unique insertion dans un decorateur global aura un effet sur toutes vos JSP.

```html

    <%@ page contentType="text/html;charset=UTF-8" language="java" %>
    <%@ taglib prefix="debug" uri="https://github.com/clescot/webappender-tag"%>

    <html>
    <head>
       <title>hello from test.jsp page</title>
    </head>
    <body>
    test.jsp
    </body>
    </html>
    <debug:webappender/>

```

Avantages de cette méthode :

- Les logs sont présentes dans le corps de la requête et non l'entête, (2Gb semble être la limite pour le corps d'une requête HTTP) ; il n'y a donc pas de réel problème de taille de logs
- cela fonctionne sur tous les navigateurs

Inconvénients de cette méthode :

- cela modifie le corps de votre requête
- il n'y a pas de visualisation soignée de ces logs contrairement aux plugins présentés précédemment
- vous ne pouvez pas filtrer vos logs du côté navigateur


# Test avec une webapp exemple

Vous pouvez tester l'application de démo, qui illustre les fonctionnalités de la librairie webappender.

### Pré-requis du test

* client [git](http://git-scm.com/).
* [java 7](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
* [maven 3](http://maven.apache.org) ou supérieur
* un navigateur web


### Installer la démo

Tapez ces commandes dans votre terminal :
```bash
    git clone git@github.com:clescot/webappender.git
    cd webappender/webappender-war-example
    mvn org.apache.tomcat.maven:tomcat7-maven-plugin:2.2:run-war -Dwebappender=true
```

Aller à l'adresse suivante `http://127.0.0.1:8080/webappender` , et visualiser les logs en fonctions du navigateur choisi, et des entêtes transmises.

# Conclusion

La librairie webappender, permet relativement simplement, de visualiser dans son navigateur, les logs générées par sa requête côté serveur.
Webappender peut être installée, sur des applications JEE utilisant logback. 










