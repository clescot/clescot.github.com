---
layout: post
title: "génération d'une image scaleway en vue d'un déploiement Terraform"
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



# génération de l'image docker

Il est nécessaire d'utiliser une image, pour que le serveur Scaleway créé via Terraform puisse exécuter un système d'exploitation.
Scaleway propose de réutiliser le format de docker (Dockerfile), avec quelques adaptations, pour définir sa propre image.

Pour cela, Scaleway fournit un repository github appelé [image builder](https://github.com/scaleway/image-builder), permettant de construire une image iso suivant ses besoins, via un serveur Scaleway.
Des images sont fournies par la communauté et validées par l'hébergeur pour être disponibles pour tous, mais on ne peut pas dire que Scaleway soit d'une grande célérité pour les valider (ubuntu et la dernière version de docker ici), ou pour les fournir.

À ce jour (avril 2017), l'image fournie ne contient que docker 1.12.2, soit 3 versions de retard...
Il est donc nécessaire de construire sa propre image.

## création du serveur permettant de construire son image

Un préalable est bien sûr d'avoir un compte Scaleway.
La création d'un serveur de construction d'image peut être fait simplement via la ligne de commande de Scaleway appelée [scw](https://github.com/scaleway/scaleway-cli).
Rappelons que la création de **cette image est spécifique à cet hébergeur**.

`scw` a été pensé pour ressembler à la ligne de commande docker. Vous retrouverez de nombreuses sous-commandes docker qui ont été reprises dans celle-ci.

Identifiez-vous au préalable sur Scaleway via scw :
`scw login` 


`scw images` vous permet de lister les images disponibles, tant celles de la communauté scaleway que les vôtres.

Vous retrouverez donc l'image dédiée à la construction d'images personnalisées appelée *image-builder*.

`scw run` vous permet de créer un serveur et le démarrer. Vous lancerez donc un nouveau serveur de création d'image via :

`scw run --name="mon-image-perso" image-builder`

À noter que la construction du serveur n'est pas instantané (quelques minutes), il faudra donc être patient.

Cette commande vous donnera de plus, une connexion ssh sur votre serveur nouvellement crée.

Sur ce serveur, exécutez :

`image-builder-configure`

puis identifiez-vous via votre identifiant (mail) et votre mot de passe.
Cette étape préalable permettra, une fois l'image construite, de pousser l'image dans l'infrastructure Scaleway pour qu'elle soit disponible (uniquement pour notre compte), lors de la création de nouveaux serveurs (le but de ce second article).

Le serveur nouvellement crée contient 2 fichiers d'exemple qui nous intéressent : `Makefile.sample` et `Dockerfile.sample`.

Créez un répertoire dédié :

`mkdir myimage`
copiez les fichiers d'exemple :

`cp Makefile.sample myimage/Makefile`

puis 

`cp Dockerfile.sample myimage/Dockerfile`

et rentrez dans le répertoire:

`cd myimage`

### Makefile 

Ce fichier permet de définir :
- le nom de l'image à construire
- la version
- le titre
- la description
- l'architecture
- la taille du volume (espace disque) associé
- le nom du script à exécuter lors du démarrage du noyau linux


Modifiez le fichier Makefile avec le descriptif, le nom et la version voulue.

```makefile

NAME =                  my-image
VERSION =               latest
VERSION_ALIASES =       1.2.3 1.2 1
TITLE =                 My image
DESCRIPTION =           My image with Ubuntu and MySQL
DOC_URL =
SOURCE_URL =            https://github.com/scaleway-community/...
VENDOR_URL =
DEFAULT_IMAGE_ARCH =    x86_64


IMAGE_VOLUME_SIZE =     50G
IMAGE_BOOTSCRIPT =      stable
IMAGE_NAME =            My image

## Image tools  (https://github.com/scaleway/image-tools)
all:    docker-rules.mk
docker-rules.mk:
        wget -qO - https://j.mp/scw-builder | bash
-include docker-rules.mk

```


### Dockerfile

Ce fichier permet de décrire les étapes d'installation de votre serveur, comme un fichier Dockerfile classique.
Néanmoins, des commentaires sont présents dans ce fichier, qui sont indispensables pour créer l'image sur plusieurs architectures : **n'y touchez pas**.




#### contenu

```dockerfile
FROM scaleway/ubuntu:amd64-16.10
# following 'FROM' lines are used dynamically thanks do the image-builder
# which dynamically update the Dockerfile if needed.
#FROM scaleway/ubuntu:armhf-16.10       # arch=armv7l
#FROM scaleway/ubuntu:arm64-16.10       # arch=arm64
#FROM scaleway/ubuntu:i386-16.10        # arch=i386
#FROM scaleway/ubuntu:mips-16.10        # arch=mips

# Prepare rootfs
RUN /usr/local/sbin/scw-builder-enter

# Add your commands here (before scw-builder-leave and after scw-builder-enter)
RUN sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
RUN sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

RUN  apt-get update && sudo apt-get install -y docker-ce

RUN mkdir -p /etc/systemd/system/docker.d
COPY docker.conf /etc/systemd/system/docker.service.d/override.conf
RUN systemctl enable docker

# Clean rootfs
RUN /usr/local/sbin/scw-builder-leave

```

#### base de l'image

Cette image dérive d'une image Ubuntu relativement récente (ici 16.10).

Un choix plus conservateur serait Ubuntu 16.04 LTS (LTS pour Long Term Support).

Notez qu'il est important de choisir une distribution ayant un noyau assez récent, car Docker s'appuie sur des fonctionnalités du noyau Linux qui se sont stabilisées que depuis peu, et évolue à la vitesse d'un cheval au galop.

Une autre alternative, qui va au bout de la démarche de "conteneurisation", serait de choisir un système d'exploitation reposant uniquement sur des conteneurs tels [rancherOS](http://rancher.com/rancher-os), [CoreOS](https://coreos.com), [Photon](https://vmware.github.io/photon) ou [Atomic](http://www.projectatomic.io). Cela fera peut-être partie d'un prochain article. L'arrivée de [LinuxKit](https://blog.docker.com/2017/04/introducing-linuxkit-container-os-toolkit), essayant d'être le dénominateur commun technique de ces initiatives, en est l'illustration.


#### description

Ce fichier Dockerfile comprend l'ajout des utilitaires nécessaires à l'ajout de la clé gpg de Docker, ainsi que le repository, puis l'installation du package `docker-ce` lui-même.

Le fichier `docker.conf` est ajouté aussi pour surcharger la configuration par défaut de docker dans systemd (en ne définissant aucune option), afin de permettre de définir par la suite ([dans le troisième article](/2017/04/29/2017-04-30-deployer-un-cluster-swarm-mode-sur-scaleway-via-terraform/)), via le fichier `/etc/docker/daemon.json` (fichier dont l'usage est maintenant recommandé par Docker), les différentes options du démon docker, dont les ports. 

Cette étape n'aurait pu être incluse dans l'image générée et donc dans ce Dockerfile, car nous incluons dans le fichier `/etc/docker/daemon.json` l'adresse IP privée du serveur crée, qui est dynamique.


`docker.conf` :

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd

```

### lancer la création de l'image

exécuter la commande suivante pour créer l'image :
`make image_on_local`

Après l'exécution de la commande précédente, Vous devriez voir votre nouvelle image (à la première ligne) dans [la liste de celles disponibles](https://cloud.scaleway.com/#/zones/par1/images).
via la commande : 

`scw images`


```
REPOSITORY                  TAG                 IMAGE ID            CREATED             REGION              ARCH
user/My_image               latest              667077c2            7 minutes           [     par1]         [x86_64]
user/My_image_With_docker   latest              fd23ff8e            2 weeks             [     par1]         [x86_64]
user/My_image_With_docker   latest              ecaf8b3b            2 weeks             [     par1]         [x86_64]
Ubuntu_Yakkety              latest              d53fafe4            6 months            [ams1 par1]         [arm x86_64]
Mattermost                  latest              0644b229            9 months            [ams1 par1]         [x86_64]
Ubuntu_Xenial               latest              656de689            12 months           [ams1 par1]         [arm x86_64]
```


Votre image apparait avec 8 caractères, c'est-à-dire un identifiant tronqué (ici `667077c2`).

Pour connaître l'identifiant complet, qui sera utilisé lors de la création des serveurs, exécutez la commande suivante :

`scw inspect 667077c2`

Celle-ci retourne :
```
[{
  "id": "667077c2-1d1b-465b-aff4-f22e58a61149",
  "name": "My image",
  "creation_date": "2017-04-22T22:39:56.884833+00:00",
  "modification_date": "2017-04-22T22:39:56.884833+00:00",
  "root_volume": {
    "id": "d586f059-3c79-470a-9511-fffaad2a8406",
    "size": 50000000000,
    "name": "x86_64-my-image-latest-2017-04-22_22:34",
    "volume_type": "l_ssd"
  },
  "default_bootscript": {
    "bootcmdargs": "LINUX_COMMON ip=:::::eth0: boot=local", 
    "initrd": "http://169.254.42.24/initrd/initrd-Linux-x86_64-v3.12.4.gz",
    "kernel": "http://169.254.42.24/kernel/x86_64-4.10.8-std-1/vmlinuz-4.10.8-std-1",
    "architecture": "x86_64",
    "id": "8fd15f37-c176-49a4-9e1d-10eb912942ea",
    "organization": "11111111-1111-4111-8111-111111111111",
    "title": "x86_64 4.10.8 std #1 (stable)",
    "public": true
  },
  "organization": "00000000-0000-5000-9000-000000000000",
  "arch": "x86_64"
}]
```
L'identifiant complet est donc `667077c2-1d1b-465b-aff4-f22e58a61149`.

L'image est prête à être utilisée pour créer nos serveurs.

## l'image n'est pas sécurisée

Veuillez noter que **cette image n'est pas du tout sécurisée**. En effet, la configuration SSH n'est pas durcie :

Voici une liste non exhaustive des manquements quant à sa sécurisaition :
- la désactivation de l'authentification par mot de passe
- le changement de port SSH
- l'activation de la séparation des privilèges
- la désactivation des algorithmes faibles
- le ssh port knocking 
- etc ...

[Une liste complète pourra être consultée sur le site de l'anssi](https://www.ssi.gouv.fr/guide/recommandations-pour-un-usage-securise-dopenssh/).

De plus, des mécanismes de sécurité ne sont pas installés (`fail2ban` par exemple).

Enfin, le fonctionnement des règles des groupes de sécurité réseau de Scaleway sont pour le moins particulières (ou j'ai loupé quelque chose) : 

Même si ces règles contiennent les plages IP depuis lesquels accepter le traffic (ici des IP internes 10.0.0.0/8 bien que cela ne nous prémunisse pas des autres clients Scaleway), les ports sont quand mêmes accessibles via l'extérieur....

En bref, la mise en place dans l'image (ce qui n'est pas le cas ici), d'un pare-feu ([ufw](https://doc.ubuntu-fr.org/ufw), ou directement netfilter par exemple), n'acceptant que des connexions des serveurs du cluster est nécessaire au delà des quelques  minutes de test de cet article.

Je regrette de plus, que scaleway ne bénéficie pas du réseau privé disponible sur l'infrastructure de la maison mère online.net (appelé RPN).


# Conclusion

La définition et la construction d'une image de serveur sont facilitées par les outils fournis par Scaleway, inspirés de ceux de Docker. Malheureusement, cette image ne sera pas utilisable directement chez d'autres hébergeurs. [Le  troisième et dernier article](/2017/04/29/2017-04-30-deployer-un-cluster-swarm-mode-sur-scaleway-via-terraform/) réutilisera l'image créée ici, pour créer des serveurs et y déployer le cluster Docker Swarm Mode.