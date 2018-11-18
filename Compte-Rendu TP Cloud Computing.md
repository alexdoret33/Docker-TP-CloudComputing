# TP Cloud

Récupération du dépôt Git Modification du Vagrantfile pour l'ajout d'un disque de 10gb et l'ajout d'une interface bridgé pour la connection Ethernet. Création de trois machines avec le Vagrantfile. Nos machines sont bien reliées avec un câble et sont bien dans le même réseau et ont pour adresse IP `172.17.8.90` et `172.17.8.91` avec un /24. Elles se Ping parfaitement bien. Nous allons répartir les services sur nos deux VMs

Nous avons créé 5 machines. 3 machines sur un hôte qui seront les managers et 2 machine sur l'autre hôte qui feront office de workers.

Les noeuds auront donc pour nom : core-01/02/03 et core-11/12
Création du fichier dans /etc/docker deamon.json avec à l'interieur 
```
{ 
    "experimental":true, 
    "metrics-addr":"0.0.0.0:9323" 
}
```

Sur core-01 (Le premier manager), nous avons exectuté cette commande: 
`docker swarm init --advertise-addr 172.17.8.101` 
Cette commande génère une commande à executer en root dans core-02 et core-03: 
`docker swarm join-token manager` 
Core-01, core-02, core-03 core-11 et core-12 ont bien rejoint le Swarm.
![](https://i.imgur.com/LuN7SSq.png)


## Déploiement d'application sur le Swarm

Nous sommes passés directement à la partie NFS car la connexion réseau était déplorable et il était trop contraignant de télécharger les images 5 fois.
Nous avons récupéré le dépôt git de l'application.

On a créé un repository:
`docker service create --name registry --publish published=5000,target=5000 registry:2`

On modifie le docker-compose.yml afin d'ajouter les images à notre repository.
```
app-proxy:
    image: localhost:5000/app-proxy:latest
#...
```
et
```
services:
  python-app:
    image: localhost:5000/python-app:latest
#...
```
On build : `docker-compose build`
On push les images : `docker-compose push`
et on déploie l'application : `docker stack deploy -c docker-compose.yml pyapp`

HOP ! L'application python est déployée sur le cluster !

## Installation et configuration de Weave Cloud

Création du compte Weave Cloud, puis nouvelle instance qui nous donne un service token. 
Ce service token est à mettre dans cette commande: `docker run -it --rm     -v /var/run/docker.sock:/var/run/docker.sock     weaveworks/swarm-agents install [service-token]`
Éxecuter ces commandes:
```
sudo curl -L git.io/scope -o /usr/local/bin/scope
sudo chmod a+x /usr/local/bin/scope
scope launch --service-token=z6n7dqmjt5cd5f4duwr5oy4xqd4byug5

Selectionner l'environnement et atendre que cela termine l'installation dans les VMs.
```
## Keepalived

Pour configurer Keepalived et rendre notre cluster accessible avec une seule adresse IP, il faut utiliser la commande suivante : 
```
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --net=host \
  -e KEEPALIVED_VIRTUAL_IPS=172.17.8.100 \ #défini une adresse virtuelle
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.17.8.101', '172.17.8.102', '172.17.8.103', '172.17.8.201', '172.17.8.202']" \ #on l'applique pour toutes les autres IP
  -e KEEPALIVED_PRIORITY=200 \
  osixia/keepalived:1.3.5
```




## NFS

Pour créer un dossier partagé, on a utilisé une machine sous centOS avec l'IP suivante : 172.17.8.150 (afin qu'elle soit sur le même réseau mais avec une IP reconnaissable)
Tout d'abord, on install l'utilitaire NFS : 
```
yum install nfs-utils
```

on crée le dossier que l'on va partager : 
```
mkdir /data
```
On change ses permissions : 
```
chmod -R 755 /data
chown nfsnobody:nfsnobody /data
```
On active et lance les services NFS à l'aide des commandes :
```
systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap
```
On configure le fichier /etc/exports pour déterminer avec qui partager le dossier. ici, tous les postes sur le réseau : 
```
/srv/data   172.17.8.*/24(rw,sync,no_root_squash,no_all_squash)
```
Puis, on relance le service et on paramètre le firewall pour les services NFS: 
```
systemctl restart nfs-server

firewall-cmd --permanent --zone=public --add-service=nfs
firewall-cmd --permanent --zone=public --add-service=mountd
firewall-cmd --permanent --zone=public --add-service=rpc-bind
firewall-cmd --reload
```

Voilà pour la machine CentOS. Ensuite, il faut paramétrer les clients (coreOS). Pour celà, on modifie le fichier conf.ign pour que le vagrantfile configure les machines au démarrage. On ajoute dans le fichier : 
```
{
        "contents": "[Unit]\nBefore=remote-fs.target\n[Mount]\nWhat=172.17.8.150:/srv/data\nWhere=data\nType=nfs\n[Install]\nWantedBy=remote-fs.target",
        "enable": true,
        "name": "data.mount"
      }
```


**Q2 : expliquez le principe d'un partage NFS, quels pourraient être ses limites dans le cas d'un swarm comme le nôtre (qui peut être amené à grandir) ?**

Un partage NFS est juste un dossier partagé entre plusieurs hôtes.
Si on ajoute de nouveaux workers ou managers au swarm, il faudra leur ajouter manuellement les autorisations d'accès au dossier partagé.

**Q3 : proposez une façon d'automatiser le déploiement cette conf NFS**

On peut tout automatiser avec le VagrantFile en ajoutant les commandes faites directement dans le fichier. Ainsi, tout se fera au déploiement de la machine.

## Show me your metrics

Là on a des metrics ! Un MAX de metrics ! On a suivi la doc sur Git et on a déployé l'app. Faut dire que le rendu est clairement plutôt stylé ! Voici quelques screens à l'appui.

![](https://i.imgur.com/gdSYcyd.png)
![](https://i.imgur.com/2FZwDqL.png)
![](https://i.imgur.com/Mgop9EF.png)

## Fin

Tu pourras trouver le VagrantFile utilisé pour configurer et déployer les VMs Workers. Celui pour les managers est sensiblement le même à priori si ce n'est qu'il déploie une machine de plus.