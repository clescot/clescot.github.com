---
layout: post
title: "kafkacat : lire et écrire dans kafka en ligne de commande sans JVM"
date: 2016-11-05 23:46
comments: true
categories: [kafkacat,kafka,big data,cli,outil]
---

Cette série d'articles essaie de vous faire découvrir des outils bien utiles autour de [Apache Kafka](https://kafka.apache.org/).

Le premier outil est [kafkacat](https://github.com/edenhill/kafkacat), un outil en ligne de commande qui permet facilement et rapidement de lire et d'écrire dans des topics kafka.

# outils fournis par apache kafka

Le projet open source [Apache Kafka](https://kafka.apache.org) propose une série d'outils pour intéragir avec cette plateforme de streaming distribuée ultra-performante :
 ceux-ci font partie intégrante du projet. 

Pour les utiliser, il est nécessaire de récupérer le projet sur github, et de lancer ces scripts qui lancent des machines virtuelles Java. Ces outils sont les plus aboutis et complets. Néanmoins, pour des besoins simples, il est pratique  d'utiliser des interfaces en ligne de commande (CLI).

 
# intérêt de kafkacat

Kafkacat est une interface en ligne de commande. Elle permet donc d'être chaînée, afin de filtrer les messages lus, rediriger le flux vers un fichier, ou vers d'autres outils Linux.

#installation

Des packages sont disponibles dans la plupart des distributions Linux.
Sous Ubuntu, il faut lancer : 

`sudo apt-get install kafkacat`

# un horizon des commandes kafkacat

## afficher l'aide

Pour afficher l'aide, il suffit de lancer :

`kafkacat -h`

## afficher la version

Pour afficher la version, il suffit de lancer :

`kafkacat -V`

## structure d'une commande kafkacat

une commande kafkacat s'exécute toujours de la façon suivante :

`kafacat -b host:port -mode -modeoptions`


Il est indispensable de spécifier évidemment un ou des brokers kafka auxquels se connecter, via l'attribut `-b`.

Celui-ci doit être complété par le mode avec les valeurs suivantes :

- `-L` pour lire les méta-données du cluster kafka (topics, nombre de partitions etc..)
- `-C` pour lire les messages 
- `-P` pour produire les messages

## options générales

les options suivantes sont présentes dans tous les modes :

- `-b` pour spécifier l'hôte et le port des brokers kafka de la forme `host1:port1;host2:port2,host3:port3`
- `-t` pour spécifier le topic
- `-c` pour limiter le nombre de message produits ou lus
- `-p` pour spécifier la partition
- `-G` pour spécifier le groupe de consommateurs
- `-X` pour spécifier des options à la librairie libdrdkafka C sous-jacente à kafkacat
- `-K` pour spécifier le délimiteur de la clé (optionnelle) d'un message
- `-D` pour spécifier le délimiteur de la valeur d'un message
- `-q` pour que kafkacat soit silencieux
- `-d` pour permettre le débugging via la librairie libdrdkafka


## lire les métadonnées du cluster


Lister les topics présent dans un broker se fera via la commande suivante :

` kafkacat -b 127.0.0.1:9092 -L`

et donnera ce type de résultat :

```bash
    Metadata for all topics (from broker 1: 127.0.0.1:9092/1):

    1 brokers:

    broker 1 at 127.0.0.1:9092

    3 topics:

    topic "Topic2" with 1 partitions:

    partition 0, leader 1, replicas: 1, isrs: 1

    topic "test-compact" with 1 partitions:

    partition 0, leader 1, replicas: 1, isrs: 1

    topic "Topic1" with 1 partitions:

    partition 0, leader 1, replicas: 1, isrs: 1
```

Indiquant :

- la liste des brokers avec leur identifiant
- la liste des topics
- pour chaque topic, les partitions
- pour chaque partition le broker hébergeant la partition leader, les brokers hébergeant les réplicas, et le nombre de réplicas synchronisés (`isrs`)
 



## lire des messages

Kafka stocke ses messages dans des topics, qui contiennent une ou plusieurs partitions. 
Les message sont toujours lus du plus ancien au plus récent, et ont un ordre garanti au sein d'une même partition.

`kafkacat -b mybroker:9092 -C -t mytopic`

Cette commande permet de lister les messages présents dans le topic `mytopic`, mais ne se terminera pas : kafkacat restera en écoute de nouveaux messages.

### lire des messages et quitter

Afin de terminer la commande après avoir lu les messages, il faut ajouter l'option `-e` -pour *exit*) :
`kafkacat -b mybroker:9092 -C -t mytopic -e`

### lire des messages depuis un offset donné

Par défaut, kafkacat lit les messages du topic depuis les offsets les plus récents de chaque partition. Néanmoins, on peut spécifier à partir de quel offset on souhaite lire le topic via l'argument `-o`.
Les valeurs possibles sont : 

- `beginning` 
- `end`
- `stored`
- `une valeur positive`
- `une valeur négative`

#### beginning

`kafkacat -b host:port -C -t mytopic -o beginning -e`

lira les messages du topic depuis le plus ancien message encore stocké pour chaque partition (le message peut disparaître plus ou moins rapidement en fonction des règles de suppression configurées dans kafka).

#### end

`kafkacat -b host:port -C -t mytopic -o end`

lira les futurs messages à partir du dernier message stocké pour chaque partition au moment de l'interrogation de kafka.

Y coupler `-e` n'aura pas trop de sens, car la commande sortira avant d'avoir pu récupérer un nouveau message.


#### stored

A la place de spécifier en option l'offset à partir duquel lire les messages, kafkacat peut lire l'offset de début de lecture pour chaque partition via une autre source de données : 
- de façon locale, via un répertoire :

`kafkacat -b mybroker.example.org  -C -t mytopic -e -o stored -c 1000 -X offset.store.method=file -X topic.auto.commit.interval.ms=250 -X topic.offset.store.path=/tmp/kafka-offsets`

Permettra de lire les messages de ce topic depuis les offsets désignés pour chaque partition. Si le répertoire est vide, kafkacat créera les fichiers nécessaires pour stocker ces offsets (de la forme `topicname-partition*.offset`). Le dernier offset lu sera sauvegardé toutes les 250 millisecondes.

- de façon distribuée dans kafka lui-même : 
`kafkacat -b mybroker.example.org  -C -t mytopic -e -o stored -c 1000 -X offset.store.method=broker -X topic.auto.commit.interval.ms=250 -X group.id=mygroup `

L'offset est stocké dans un topic spécial, utilisé par kafka pour connaître les derniers offsets de chaque partition de chaque topic, pour chaque groupe de consommateurs.
Il est donc nécessaire de passer à kafka l'identifiant du groupe de consommateurs pour qu'il puisse identifier quel dernier message vous avez lu.

#### valeur positive

`kafkacat -b host:port -C -t mytopic -o 31559 -e`

lira les messages depuis l'offset 31559 de toutes les partitions jusqu'à la fin du topic au moment de l'interrogation de kafka.


#### valeur négative

Il est possible de lire (approximativement), par exemple les 1500 derniers messages d'un topic (sur l'ensemble des partitions), en lançant la commande suivante :

`kafkacat -b host:port -C -t mytopic -o -1500 -e`


### lire des messages sur une partition donnée

Il est possible de lire sur une partition donnée via `-p`, avec la commande suivante :

`kafkacat -b host:port -C -t mytopic -o -1500 -p 4 -e`

Ne seront retournés que les messages du topic stockés dans la partition numéro 4.

### lire des messages avec un groupe de consommateurs donné

Il est possible de lire des messages en tant que membre d'un groupe de consommateurs avec la commande suivante :

`kafkacat -b host:port -C -t mytopic -G mygroup -e`

Kafka nous assignera des partitions du topic à lire (toutes les partitions si nous sommes le seul membre actif du groupe), et nous délivrera les messages à partir des derniers offsets lus pour chaque partition.



### lire des messages avec un format de sortie personnalisé


Les options de formattage sont :

- `%t` pour le topic
- `%p` pour la partition
- `%o` pour l'offset
- `%k` pour la clé
- `%S` pour la taille du message
- `%s` pour le cotnenu du message

`kafkacat -b broker:port -t mytopic -e -f 'Topic %t[%p], offset: %o, key: %k, payload: %S bytes: %s\n' `

### lire des messages avec une enveloppe JSON  en sortie

L'option `-J` permet d'avoir une enveloppe JSON encadrant la clé, et le message.

`kafkacat -b host:port -C -t mytopic -e -J `



## écrire des messages


### écrire des messages depuis la sortie standard 

Pour écrire un message depuis la sortie standard :

`kafkacat -b 127.0.0.1:9092 -t Topic2 -P -c 1 -e`

écrire son message et appuyez sur la touche entrée pour terminer le message.

### écrire des messages depuis un fichier contenant plusieurs messages

Il est possible de produire des messages depuis un fichier les contenant délimités par des caractères séparateurs, via l'argument `-l`.

`kafkacat -b 127.0.0.1:9092 -t Topic2 -P -e -l file1`

Cette option n'est possible que pour un fichier.

### écrire des messages depuis des fichiers

Il est possible de produire des messages depuis plusieurs fichiers, chacun contenant un seul message.
Même si ces fichiers contiennent des séparateurs, le contenu de chaque fichier sera considéré comme un seul message.

`kafkacat -b 127.0.0.1:9092 -t Topic2 -P -e file1 file2 file3`


### écrire des messages depuis des fichiers via des séparateurs personnalisés

Il est possible de définir les séparateurs de messages via l'option `-D`, et les séparateurs des clés des messages via l'option `-K` :

`kafkacat -b 127.0.0.1:9092 -t Topic2 -P -e -D% -K/ file1 file2 file3`

Un fichier pourra avoir la structure suivante :

`7892/monmessage%7893/deuxiememessage%7894/troisiememessage`

# compatibilité des versions de kafka avec kafkacat


Comme évoqué auparavant, kafkacat repose sur la librairie C [libdrdkafka](https://github.com/edenhill/librdkafka).

Certaines versions de cette librairie ont pour comportement par défaut d'intéragir avec le protocole de kafka dans une certaine version;
ce comportement par défaut ne correspond pas toujours à la version de kafka que vous utilisez. Il faut donc spécifier via des paramètres libdrdkafka (option `-X`)le protocole à utiliser. Ainsi, quand on lit un kafka 0.10.1.0 avec kafkacat 1.3.0-1 on peut obtenir l'erreursuivante :


```bash
%4|1480463638.946|PROTOERR|rdkafka#consumer-1| 127.0.0.1:9092/9092: Protocol parse failure at rd_kafka_fetch_reply_handle:3864 (incorrect broker.version.fallback?)
%4|1480463638.946|PROTOERR|rdkafka#consumer-1| 127.0.0.1:9092/9092: expected 10370775 bytes > 44 remaining bytes
%4|1480463639.551|PROTOERR|rdkafka#consumer-1| 127.0.0.1:9092/9092: Protocol parse failure at rd_kafka_fetch_reply_handle:3864 (incorrect broker.version.fallback?)
%4|1480463639.551|PROTOERR|rdkafka#consumer-1| 127.0.0.1:9092/9092: expected 10370775 bytes > 44 remaining bytes
%4|1480463640.156|PROTOERR|rdkafka#consumer-1| 127.0.0.1:9092/9092: Protocol parse failure at rd_kafka_fetch_reply_handle:3864 (incorrect broker.version.fallback?)
```


Le paramétrage suivant permettra de résoudre ce problème : 

`kafkacat -b mybroker:9092 -C -t mytopic -e -X api.version.request=true`

Les compatibilités des versions et les options à activer en fonction sont listées dans la page suivante :

[page de compatibilité](https://github.com/edenhill/librdkafka/wiki/Broker-version-compatibility)



# limites de kafkacat

Kafkacat ne permet pas toutes les intéractions permises par les outils intégrés dans le projet Kafka.

La principale cause est l'absence de toutes les fonctionnalités de ces outils dans le protocole d'interaction avec Kafka.
De nombreuses avancées arrivent de version en version, mais des manques persistent.

## création et suppression de topics

Kafka **0.10.1.0** intègre dans son protocole **la création et la suppression de topics**.
Kafkacat **1.3.0** n'intègre pas cette fonctionnalité qui est prévue pour la version **1.4.0**.


## recherche par timestamp 

Cette fonctionnalité est aussi apparue dans la version **0.10.1.0** de kafka. Elle n'est pas encore supportée par kafkacat.

## support de la compression lz4 

La compression **lz4**, ajoutée en complément des compressions **gzip** et **snappy**, a été implémentée dans kafkacat mais n'est pas encore présent dans le paquet distribué par ubuntu.

Il conviendra donc de compiler depuis les sources kafkacat, avec une version plus récente de la librairie **libdrdkafka** pour supporter ce nouvel algorithme.

# conclusion

Kafkacat me semble être un bon outil en ligne de commande, permettant d'interagir de façon performante avec kafka, et de s'intégrer aux autres lignes de commandes Linux via le chaînage des commandes.

Néanmoins, il s'avère dans certains cas indispensable d'avoir une version récente de kafkacat, ou de passer par les outils fournis avec kafka.
