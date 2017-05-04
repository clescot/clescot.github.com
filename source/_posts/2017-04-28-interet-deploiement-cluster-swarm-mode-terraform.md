---
layout: post
title: "intérêts du déploiement d'un cluster Docker Swarm mode via Terraform"
date: 2017-04-29 12:45
comments: true
categories: [docker,docker,swarm mode,scaleway,terraform,devops, ubuntu]
---


# Contexte

Je vous propose dans cette série d'articles, d'installer un cluster docker Swarm Mode via Terraform chez l'hébergeur Scaleway.

Celle-ci comprend 3 articles :

- [l'intérêt de cette solution (ne contient pas d'éléments techniques)](/2017/04/29/2017-04-28-interet-deploiement-cluster-swarm-mode-terraform/)
- [la génération de l'image des serveurs Scaleway à déployer](/2017/04/29/2017-04-29-generation-image-scaleway-terraform/)
- [le déploiement via Terraform du cluster Docker Swarm Mode](/2017/04/29/2017-04-30-deployer-un-cluster-swarm-mode-sur-scaleway-via-terraform/) 

# Caractéristiques générales de la solution

Dans ce premier article, voyons les caractéristiques de cette solution (n'hésitez pas à passer directement [au deuxième article si vous êtes allergiques au bla-bla](/2017/04/29/2017-04-29-generation-image-scaleway-terraform/)) :
il s'agit d'une infrastructure auto-descriptive, flexible, immutable, à forte densité, résistante aux pannes.


## une infrastructure auto-descriptive

Au lieu d'opérer des opérations d'installation et de configuration, de façon manuelle, et non documentée, il est indispensable de nos jours, d'utiliser des outils automatisés, reposant sur des informations textuelles, que l'on peut gérer via des systèmes de gestion de version ([git](https://git-scm.com) par exemple).
La seule lecture de fichiers doit pouvoir décrire dans son intégralité la solution mise en place.

L'infrastructure déployée doit être automatisée via du code (bye-bye les clickodromes non reproductibles).
Cette tendance est appelée `infrastructure as code` en Anglais.

## une infrastructure immutable

L'"immutabilité" (en vrai français l'immuabilité), est particulièrement à la mode en ce moment, tant au niveau de la programmation (Erlang, Scala ...), que des infrastructures.

Concernant la fourniture de serveurs, dès qu'un changement doit être opéré, au lieu de modifier celui-ci, nous allons le détruire pour en recréer un autre, de façon automatisée. Cela évitera dans le temps, de ne plus maîtriser l'état du serveur ; en effet, l'installation, la mise à jour, la suppression, puis la réinstallation d'une nouvelle version d'un logiciel sur un serveur n'est souvent pas équivalent à l'installation directe de cette nouvelle version. L'état du serveur devient de moins en moins maîtrisable au cours du temps, il se dégrade.

Ainsi, des outils forts pratiques tels [Ansible](https://www.ansible.com), [Puppet](https://puppet.com) ou [Chef](https://www.chef.io/chef/),  pour ne citer que les plus connus, qui automatisent beaucoup de tâches, sont de moins en moins utilisés pour déployer des infrastructures (serveurs, DNS, réseaux etc..), car leur utilisation répétée sur ces mêmes ressources ne permet plus de maîtriser précisément leurs états au bout d'un certain temps. Néanmoins, ils restent présents concernant la finalisation quelquefois complexe des serveurs.

Cette évolution est traduite en anglais par l'expression "pets vs cattle", c'est-a-dire la comparaison de serveurs à des animaux de compagnie ("pets"), auquel on porte une grande attention (pour faire le parallèle à la grande complexité actuelle pour maîtriser l'état de nos serveurs), à opposer au bétail ("cattle"), auquel on n'attache pas une grande importance (on n'hésite pas à recréer un serveur au lieu de le faire évoluer). 


## une infrastructure flexible

Concernant la flexibilité, l'appel d'APIs de fournisseurs d'infrastucture à la demande (IAAS ou infrastructure as a service) s'impose de plus en plus.
Les hébergeurs peuvent être internes aux grandes entreprises avec des déploiements de centre de données VMWare par exemple, ou externes avec des fournisseurs tels [Amazon Web Services (AWS)](http://aws.amazon.com), Digital Océan, ou Scaleway dans cette série d'articles.
Cette flexibilité permet d'ajuster son infrastructure en temps réel au gré de ses besoins.

## une infrastructure à haute densité

L'usage tendant à se généraliser des conteneurs, principalement [Docker](https://www.docker.com), permet de faciliter et standardiser l'installation de logiciels, et de densifier les serveurs.
En effet, l'installation de multiples conteneurs sur un même serveur, permet de rentabiliser au mieux l'usage des serveurs : de nombreux logiciels se côtoient sans effets de bords notables (mise à part une concurrence accrue sur l'usage des ressources serveurs). 

## infrastructure résistante aux pannes

Une infrastructure résistante au pannes sera distribuée et redondée sur plusieurs serveurs (voire plusieurs centre de données), afin de pallier à une défaillance de l'un d'entre eux.


# et concrètement du côté de la technique ?


## Docker Swarm Mode
L'usage de conteneurs Docker tend à se généraliser pour standardiser le déploiement d'applications.
Les différents conteneurs devant intéragir pour former des applications (serveur web, base de données etc...), différentes solutions d'orchestration existent.
J'ai choisi pour cet article une solution récente et simple : 
[docker swarm mode](https://docs.docker.com/engine/swarm).

Elle n'est pas aussi mûre que [Mesos](http://mesos.apache.org/), ni la plus complète ([Kubernetes](https://kubernetes.io)), mais est une des plus simples (avec [Cattle](https://github.com/rancher/cattle) un des orchestrateurs supportés par [Rancher](http://rancher.com)).

## Terraform

Nous utiliserons [Terraform](https://www.terraform.io), une solution de description et de construction d'infrastructure immutable. Celle-ci comprend de nombreux fournisseurs (providers), dont scaleway pour les serveurs, et cloudflare pour le DNS.

## Scaleway
J'ai choisi l'hébergeur français [Scaleway](http://www.scaleway.com), filiale de [Online](www.online.net), pour son coût modique, et la présence d'un "provider" Terraform.
Cette solution est encore assez jeune, et fournit un niveau très basique d'équipement à la demande, contrairement à sa maison mère (pas de réseau privé virtuel RPN ici par exemple).

## Cloudflare

J'ai choisi d'utiliser cloudflare pour gérer le nom de domaine, car celui-ci fournit ce service gratuitement, et permet d'inclure une étoile (wildcard), comme sous-domaine, permettant que tous les sous-domaines soient redirigés vers le cluster Docker Swarm Mode.

# Conclusion

Les différentes propriétés décrites dans ce premier article, permettent je l'espère de vous faire comprendre l'intérêt des outils mis en oeuvre.

Passons maintenant à la lecture du [deuxième article ](/2017/04/29/2017-04-29-generation-image-scaleway-terraform/) pour passer à la mis en oeuvre.