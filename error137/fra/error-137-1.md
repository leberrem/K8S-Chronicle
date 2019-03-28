# Il était un fois

Cette histoire commence par la mise en place d'un cluster Kubernetes et la mise en place de GITLAB Community Edition.
A l'activation de la CI/CD avec rattachement du cluster par "simple" configuration, nous obtenons un namespace dédié avec un process runner disponible pour lancer nos traitements.

Pour planter le décors on a un cluster Kubernetes avec 3 nodes master, 3 nodes worker.
Chaque noeud worker possedant 2 volumes rattachés pour du provisionning local et du provisionning distribué via ROOK CEPH.
Une instance indépendant pour GITLAB et enfin un bastion pour accéder à tout ça.
On automatise le tout avec terraform et ansible.

Voici un petit schéma très simplifié.

![cluster](/error137/img/cluster.png)

Bon là on se dit qu'on a un truc qui devrait tenir le route pour les premiers tests et normallement les DEVs devraient être content.

Sauf que le cluster commence rapidement à devenir instable surtout lors des opérations de CI/CD.

Et c'est là que l'histoire commence vraiment.

A chaque fois qu'ils ont un plantage, ils ont une erreur `137`

![error137](/error137/img/error137.png)

Cette erreur semble aléatoire et don cnon maitrisé.
Il ne me faut pas plus pour piquer ma curiosité et tenter de comprendre ce qui se cache sous le capot.

[< Précédent](error-137-0.md) | [Sommaire](error-137-0.md) | [Suivant >](error-137-2.md)