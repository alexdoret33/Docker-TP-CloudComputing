Compte-Rendu TP Cloud Computing

Récupération du dépôt Git
Modification du Vagrantfile pour l'ajout d'un disque de 10gb et l'ajout d'une interface bridgé pour la connection Ethernet.
Création de trois machines avec le Vagrantfile.
Nos machines sont bien reliées avec un câble et sont bien dans le même réseau et ont pour adresse IP 192.168.10.1 et 192.168.10.2 avec un /24.
Elles se Ping parfaitement bien.
Nous allons répartir les services sur nos deux VMs

Création du fichier dans /etc/docker deamon.json avec à l'interieur 
{
        "experimental":true,
        "metrics-addr":"0.0.0.0:9323"
}
Sur core-01 (Le premier manager), nous avons exectuté cette commande: docker swarm init --advertise-addr 172.17.8.101
Cette commande génère une commande à executer en root dans core-02 et core-03: docker swarm join-token manager
Core-01, core-02, core-03 ont bien rejoint le Swarm

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
3z55bvd0wf69v4hkpq3kyx30r *   core-01             Ready               Active              Leader              18.06.1-ce
7mt26olfko7cyu9n8oaluxraz     core-01             Down                Active                                  18.06.1-ce
chg9wqxeffvkblb5z7ck6m39l     core-02             Down                Active                                  18.06.1-ce
m2174yj1go9zgh33ooapjlur6     core-02             Ready               Active              Reachable           18.06.1-ce
t8viclsxhpu6bzuan35856pkk     core-03             Ready               Active              Reachable           18.06.1-ce
zlydhywkhetjywqrt4hqt1d0a     core-11             Ready               Active                                  18.06.1-ce
ing3elcjtwvvglbawy0nzb6g5     core-12             Ready               Active                                  18.06.1-ce

Nous sommes passés directement à la partie NFS car la connexion réseau était déplorable.

Nous avons 