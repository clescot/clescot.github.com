---
layout: post
title: "déployer un cluster Docker Swarm mode sur Scaleway via Terraform"
date: 2017-04-30 12:45
comments: true
categories: [docker,docker,swarm mode,scaleway,terraform,devops, ubuntu]
---

# Contexte

Je vous propose dans cette série d'articles, d'installer un cluster docker Swarm Mode via Terraform chez l'hébergeur Scaleway.

Celle-ci comprend 3 articles :

- [l'intérêt de cette solution (ne contient pas d'éléments techniques)](/2017/04/29/2017-04-28-interet-deploiement-cluster-swarm-mode-terraform/)
- [la génération de l'image des serveurs Scaleway à déployer](/2017/04/29/2017-04-29-generation-image-scaleway-terraform/)
- [le déploiement via Terraform du cluster Docker Swarm Mode](/2017/04/29/2017-04-30-deployer-un-cluster-swarm-mode-sur-scaleway-via-terraform/)

# Déploiement via Terraform du cluster Docker Swarm Mode 

## organisation des fichiers terraform

Les fichiers [terraform](https://www.terraform.io), (qui regroupent les ressources de l'infrastructure voulue), sont les suivants :

### terraform.tf

Il s'agit du fichier de configuration principal. Il décrit l'infrastructure voulue, sous forme de ressources.
Ces ressources sont fournies par des [providers](https://www.terraform.io/docs/providers/index.html), et mis en place avec l'aide de [provisioners](https://www.terraform.io/docs/provisioners/index.html) afin de déployer des fichiers sur les serveurs, ou d'exécuter des commandes (locales au poste qui lance le déploiement ou distantes sur les serveurs cibles).

le fichier est organisé via les sections suivantes :

- définition des providers (ici scaleway et cloudflare)
- définition de la ressource "*manager_init*", qui correspond au premier serveur scaleway servant à initialiser le cluster swarm (un noeud *manager*) :
 Ce premier serveur initiant le cluster est nécessaire ; il génère les tokens de sécurité pour les autres noeuds ayant le rôle de *manager* ou *worker*.

La récupération des tokens de sécurité Docker Swarm mode se fait via des appels distants vers le premier Manager sur le port 2375.

Une solution plus sûre pourrait passer plutôt par l'écriture du token sur un fichier sur le premier serveur lors de sa génération, une récupération en locale, puis un upload en scp sur les serveurs nécessitant ces tokens lors de la connexion à ceux-ci juste après leur création via des *provisioners*  Terraform.

Veuillez noter aussi que le cluster Docker Swarm mode écoute sur le port 2375, qui n'est pas sécurisé. Une installation plus sûre utiliserait le port 2376 couplé à une authentification cliente par certificat SSL supportée nativement par le Docker Engine. Cette configuration dépasse largement le cadre de cet article.

- définition des autres *managers* du cluster :
 il est recommandé, pour avoir un cluster robuste, d'avoir au moins 3 *managers* présents. Une plus grande robustesse aurait induit la répartition des noeuds sur plusieurs centres de données (Scaleway propose un centre de données français, et un centre de données néerlandais pour l'instant).
- définition des *workers* du cluster
- définition de l'IP statique de référence et association au serveur d'initialisation :

Cela permet que le DNS pointe toujours vers une IP valide, même si le serveur associé change.
À noter qu'un problème persiste ici : lorsque le serveur tombe, il n'y a pas encore de mécanisme de rétro-contrôle qui associe l'IP avec un autre serveur en vie du cluster.
- ajout d'un sous-domaine wildcard (*) au DNS, pointant vers l'IP statique
Cela permet au cluster de gérer des sous-domaines de façon autonome, via [traefik](http://traefik.io) par exemple
- définition des groupes de sécurité, avec leur règles associées :
Comm indiqué précédemment,  cela n'apporte qu'une sécurité limitée, au vu du fonctionnement des groupes de sécurité de Scaleway.

Voici le contenu du fichier terraform.tf : 

```
provider "scaleway" {
  organization = "${var.scaleway_organization}"
  token = "${var.scaleway_token}"
  region = "${var.scaleway_region}"
}


provider "cloudflare" {
  email = "${var.cloudflare_email}"
  token = "${var.cloudflare_token}"
}


resource "scaleway_server" "manager_init" {
  count = 1
  name = "swarm-manager-${count.index + 1}"
  image = "${var.ubuntu_x86_64_image}"
  type = "${var.scaleway_type}"
  dynamic_ip_required = "true"
  security_group="${scaleway_security_group.internal.id}"



  provisioner "remote-exec" {
      inline = [
          "mkdir -p /etc/systemd/system/docker.d",
        "mkdir -p /etc/docker"]
  }



  provisioner "file" {
    source = "daemon_manager.tpl"
    destination = "/etc/docker/daemon_manager.tpl"
  }




  provisioner "remote-exec" {
    inline = [
      "sed -e 's/SWARM_MANAGER_PRIVATE_IP/${self.private_ip}/g' /etc/docker/daemon_manager.tpl > /etc/docker/daemon.json",
      "systemctl daemon-reload",
      "systemctl restart docker",
      "docker swarm  init --advertise-addr ${self.private_ip} --listen-addr ${self.private_ip}:2377 "
    ]
  }
}


resource "scaleway_server" "manager_join" {
  count = 2
  name = "swarm-manager-${count.index + 2}"
  image = "${var.ubuntu_x86_64_image}"
  type = "${var.scaleway_type}"
  dynamic_ip_required = "true"
  security_group="${scaleway_security_group.internal.id}"

  provisioner "remote-exec" {
    inline = [
      "mkdir -p /etc/systemd/system/docker.d",
      "mkdir -p /etc/docker"
    ]
  }


  provisioner "file" {
    source = "daemon_manager.tpl"
    destination = "/etc/docker/daemon_manager.tpl"
  }



  provisioner "remote-exec" {
    inline = [
      "sed -e 's/SWARM_MANAGER_PRIVATE_IP/${self.private_ip}/g' /etc/docker/daemon_manager.tpl > /etc/docker/daemon.json",
      "systemctl daemon-reload",
      "systemctl restart docker",
      "docker swarm join  ${scaleway_server.manager_init.0.private_ip}:2377 --token $(docker -H ${scaleway_server.manager_init.0.private_ip}:2375 swarm join-token -q manager)"
    ]
  }

  depends_on = [
    "scaleway_server.manager_init"
  ]
}




resource "scaleway_server" "worker" {
  count = 2
  name = "swarm-worker-${count.index + 1}"
  #Docker
  image = "${var.ubuntu_x86_64_image}"
  type = "${var.scaleway_type}"
  dynamic_ip_required = "true"
  security_group="${scaleway_security_group.internal.id}"


  provisioner "remote-exec" {
    inline = [
      "mkdir -p /etc/systemd/system/docker.d",
      "mkdir -p /etc/docker"
    ]
  }


  provisioner "file" {
    source = "daemon_worker.json"
    destination = "/etc/docker/daemonjson"
  }



  #connection to the first manager to get the cluster token
  provisioner "remote-exec" {
    inline = [
      "systemctl daemon-reload",
      "systemctl restart docker",
      "docker swarm join  ${scaleway_server.manager_init.0.private_ip}:2377 --token $(docker -H ${scaleway_server.manager_init.0.private_ip}:2375 swarm join-token -q worker)"
    ]
  }

  depends_on = [
    "scaleway_server.manager_init"
  ]


}

resource "scaleway_ip" "external_ip" {
  server = "${scaleway_server.manager_init.0.id}"
}


# Add a record to the domain
resource "cloudflare_record" "domain" {
  domain = "${var.cloudflare_domain}"
  name = "*"
  value = "${scaleway_ip.external_ip.ip}"
  type = "A"
  ttl = 3600
}




resource "scaleway_security_group" "internal" {
  name = "cluster_inside"
  description = "security group to configure access from inside the cluster"
}


#accept in
variable "input_ports_from_cluster" {
  #TCP port 2377 for cluster management communications
  #TCP and UDP port 7946 for communication among nodes
  #TCP and UDP port 4789 for overlay network traffic

  type = "list"
  default = [2377,4789,7946]
}


resource "scaleway_security_group_rule" "internal_in_accept_2375" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "TCP"
  port = "2375"
}



resource "scaleway_security_group_rule" "internal_in_accept_2376" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "TCP"
  port = "2376"
}



#TCP port 2377 for cluster management communications
resource "scaleway_security_group_rule" "internal_in_accept_2377" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "TCP"
  port = "2377"
}

#TCP and UDP port 4789 for overlay network traffic
resource "scaleway_security_group_rule" "internal_in_accept_overlay_tcp" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "TCP"
  port = "4789"
}



#TCP and UDP port 4789 for overlay network traffic
resource "scaleway_security_group_rule" "internal_in_accept_overlay_udp" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "UDP"
  port = "4789"
}




#TCP and UDP port 7946 for communication among nodes
resource "scaleway_security_group_rule" "internal_in_accept_inter_nodes_tcp" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "TCP"
  port = "7946"
}


resource "scaleway_security_group_rule" "internal_in_accept_inter_nodes_udp" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "UDP"
  port = "7946"
}



#If you are planning on creating an overlay network with encryption (--opt encrypted),
#you will also need to ensure protocol 50 (ESP) is open.
resource "scaleway_security_group_rule" "internal_in_accept_esp_encryption" {
  security_group = "${scaleway_security_group.internal.id}"
  action = "accept"
  direction = "inbound"
  ip_range = "10.0.0.0/8"
  protocol = "TCP"
  port = "50"
}





resource "scaleway_security_group" "external" {
  name = "cluster_outside"
  description = "security group to configure access from outside the cluster"
}

variable "input_ports_from_external" {
  #If you are planning on creating an overlay network with encryption (--opt encrypted),
  #you will also need to ensure protocol 50 (ESP) is open.
  type = "list"
  default = [80,443,22]
}


resource "scaleway_security_group_rule" "external_in_accept" {
  security_group = "${scaleway_security_group.external.id}"

  action = "accept"
  direction = "inbound"
  ip_range = "0.0.0.0/0"
  protocol = "TCP"
  port = "${element(var.input_ports_from_external, count.index)}"
  count = "${length(var.input_ports_from_external)}"
}


#drop in
resource "scaleway_security_group_rule" "any_in_drop" {
  security_group = "${scaleway_security_group.external.id}"

  action = "drop"
  direction = "inbound"
  ip_range = "0.0.0.0/0"
  protocol = "TCP"
}

```

### daemon_manager.tpl

Ce fichier est un template pour configurer le démon docker des noeuds *manager* du cluster. la variable `SWARM_MANAGER_PRIVATE_IP` sera remplacée au démarrage pour l'adresse IP de l'interface réseau **privée** du serveur.



```json
{
"experimental" : true,
"storage-driver" : "overlay2",
"labels" : ["provider=scaleway"],
"mtu": 1500,
"hosts": ["unix:///var/run/docker.sock","tcp://SWARM_MANAGER_PRIVATE_IP:2375"]
}

```

### daemon_worker.json

Ce fichier permet de configurer le démon des noeuds *worker* du cluster Docker Swarm mode. Il ne contient pas l'adresse IP 
 de l'interface réseau privée du serveur, car les noeuds *worker* ne peuvent piloter le cluster (sauf si on les transforme en manager).

```json
{
"experimental" : true,
"storage-driver" : "overlay2",
"labels" : ["provider=scaleway"],
"mtu": 1500,
"hosts": ["unix:///var/run/docker.sock"]
}

```

### variables.tf

Le fichier `terraform.tf`, fait référence à des variables. Celles-ci sont décrites dans le fichier `variables.tf`.


```
variable "scaleway_organization" {
  description = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
variable "scaleway_token" {
  description = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
variable "scaleway_region" {
  description = "scaleway datacenter 'par1' for paris(France) or 'ams1' fr Amsterdam (Nederlands)"
  default="par1"
}
variable "scaleway_type" {
  description = "type of server"
  default="C2S"
}
variable "cloudflare_email" {
  description = "your email scaleway account identifier"
}
variable "cloudflare_token" {
  description = ""
}
variable "cloudflare_domain" {
  description = "your DNS domain managed by cloudflare"
}
variable "ubuntu_x86_64_image" {
  description = "your custom image ID"
}

```


### terraform.tfvars

Ce fichier contient les valeurs des variables référencées dans le fichier `terraform.tf`, et définies dans le fichier `variables.tf`.


```
scaleway_organization = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
scaleway_token = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
scaleway_region = "par1"
scaleway_type = "C2S"
cloudflare_email = "myname@email.com"
cloudflare_token = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
cloudflare_domain = "mondomaine.com"
ubuntu_x86_64_image = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```


### terraform.tfstate

Ce fichier est généré par Terraform, et contient l'état de l'infrastructure après application des changements.


### déclenchement de Terraform

- créer un plan d'exécution

l'exécution de la commande `terraform plan`, permet d'inspecter l'infrastructure actuelle, et l'infrastructure cible, 
afin de générer un plan d'exécution des changements nécessaires.

-  créer l'infrastructure

Cette création se fait via la commande :
`terraform apply`

- détruire l'infrastructure

La destruction se fait via la commande `terraform destroy` (après confirmation via l'invite de commandes).



# résultat


Après avoir crée via `terraform apply`, votre infrastructure (et patienté quelques minutes), en se connectant sur un des noeuds ayant le rôle de manager, vous pouvez lancer la commande suivante :

```
root@swarm-manager-1:~# docker node ls
ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
b37dxtsbn573m4xhas0j7b4do *  swarm-manager-1  Ready   Active        Leader
cvv0iyfy7tybsev0k55iyif4i    swarm-manager-2  Ready   Active        Reachable
llrhva62q0byzhruu6lu29gyx    swarm-worker-2   Ready   Active        
p4mcus11v15can4ut0lqajfqd    swarm-worker-1   Ready   Active        
pi2wxsmibs64y9d4xwnuxvew8    swarm-manager-3  Ready   Active        Reachable
```

Vous pouvez donc constater la bonne mise en place de votre cluster Docker Swarm Mode.

# conclusion

Cet article vous permet de vous familiariser avec Terraform, un outil de plus en plus présent pour automatiser la création de vos infrastructures immutables via du code.
Il vous permet aussi de vous essayer à la création d'un cluster Docker Swarm Mode.
J'attire néanmoins votre attention sur la non-sécurisation de l'image et du réseau mis en place.

Une mise en production d'un environnement de ce type, demanderait au préalable, d'autres opérations de sécurisation qui sortent du cadre de cet article.
