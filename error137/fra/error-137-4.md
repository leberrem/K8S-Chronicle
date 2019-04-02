# La transposition dans Kubernetes

## Test mémoire

Par défaut Kubernetes prévoit la définition de limites par défaut. Ceci se généralement au niveau du namespace.

https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/


## Test CPU

## Test disque

### Filesystem

### Log

Par défaut les logs sont consultables par la commande `kubectl logs`.
C'est simple ça nous masque les noeuds et en plus on peut voir les logs d'un POD et plus précisement d'un conteneur.
On peut même voir avec l'option `--previous`, les logs d'un conteneur précédement tombé, sauf dans un seul cas : L'eviction
Si on revient à notre petite histoire. c'est justement notre cas. Donc on ne peut pas s'appuyer sur cette fonctionnalité pour comprendre notre problème.

Si on regarde du coté de la rotation. Kubernetes n'implémente rien nativement, il faut intervenir directement sur les conteneurs , la daemon des conteneurs ou un outil externe. Il faut aussi penser que la commande `kubelet logs` ne connait que le dernier fichier de rotation. Ca peut aider lorsqu'on ne trouve pas de trace, ce qui peut être normal durant à la phase de rotation.

Le meilleur moyen bien evidement reste l'implémentation d'un daemonset de type fluentd qui va s'occuper de la collecte, la rotation et la redirection des logs vers un outil externe (elasticsearch, graylog, ...)

[< Précédent](error-137-3.md) | [Sommaire](error-137-0.md) | [Suivant >](error-137-5.md)