# Il était un fois

Cette histoire commence par la mise en place d'un cluster Kubernetes et la mise en place de GITLAB Community Edition.
A l'activation de la CI/CD avec rattachement du cluster par "simple" configuration, nous obtenons un namespace dédié avec un process runner disponible pour lancer nos traitements.

Pour planter le décors on a un cluster Kubernetes avec 3 nodes master, 3 nodes worker.
Chaque noeud worker possedant 2 volumes rattachés pour du provisionning local et du provisionning distribué via ROOK CEPH.
Une instance indépendant pour GITLAB et enfin un bastion pour accéder à tout ça.
On automatise le tout avec terraform et ansible.
C'est génial.

