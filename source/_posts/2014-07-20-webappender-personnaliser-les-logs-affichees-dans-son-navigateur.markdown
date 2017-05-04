---
layout: post
title: "webappender : personnaliser les logs affichées dans son navigateur"
date: 2014-07-20 11:54
comments: true
categories: [logback,web,logs,debug,communication]
---

Cette série de deux articles décrit le fonctionnement de la librairie de visualisation de logs [webappender](https://github.com/clescot/webappender). 

Le premier article expose sa mise en place, et les différentes façons de visualiser les logs suivant son navigateur.

Le deuxième article décrit la façon de choisir le contenu et la taille des logs à afficher.

# Personnaliser le contenu des logs

La librairie [webappender](https://github.com/clescot/webappender), permet de personnaliser le contenu de chaque log. par défaut, toutes les informations disponibles via logback sont incluses.

Néanmoins, certaines informations peuvent être longues à récupérer. Webappender permet donc de ne pas aller chercher et inclure ces informations, si le header HTTP `X-wa-verbose-logs=false` est transmis.

Ainsi, seront **exclues** de chaque log, les informations suivantes :

* le numéro de ligne
* le chemin du fichier hébergeant la classe java
* le temps
* le temps relatif
* le nom du thread
* la classe appelante
* la méthode appelante
* le [Mapped Diagnostic context](http://logback.qos.ch/manual/mdc.html) (MDC)
* le ThrowableProxy (permettant de récupérer les informations d'une exception)
* [le nom du contexte](http://logback.qos.ch/manual/configuration.html#contextName)
* les données du code appelant (CallerData), c'est-à-dire la stackTrace
* [le marqueur](http://www.slf4j.org/api/org/slf4j/Marker.html) 

Avec cette configuration, le temps d'exécution des requêtes est donc moins impactés lorsque le webappender est activé.

# Filtrer les logs

Webappender, supporte [les filtres logback](http://logback.qos.ch/manual/filters.html), pour réduire les traces applicatives (logs) transmises à votre navigateur.

Veuillez noter que ces filtres ne sont appliqués que dans le contexte de webappender, c'est-à-dire dans les informations transmises dans la réponse HTTP. 

Ce filtrage ne s'applique, comme toujours, qu'aux traces liées aux requêtes faites par votre navigateur.
Les traces ne sont pas filtrées dès leur création, comme pourrait le faire les [filtres turbo](http://logback.qos.ch/apidocs/ch/qos/logback/classic/turbo/class-use/TurboFilter.html).