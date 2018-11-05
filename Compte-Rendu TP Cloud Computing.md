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


## déploiement d'application sur le swarm

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
