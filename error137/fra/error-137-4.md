# La transposition dans Kubernetes

## Test mémoire

Par défaut Kubernetes prévoit la définition par défaut des limites unitaires par conteneurs. Ceci se fait au niveau du namespace. Bien evidement cette limite peut être surchargée à la création de chaque conteneur mais ne pourra exceder les valeurs man/min définies sur le namespace.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    max:
      memory: 1Gi
    min:
      memory: 128Mi
    type: Container
```

Si on créé un conteneur sans définition de limite il prendra donc les valeurs par défaut du namespace.

Il est possible de définir des min/max au niveau POD ce qui aura pour impact de contrôler à la création la somme des limites demandées pour chacun des conteneurs du POD.

On peut aussi définir ques quotas ce qui permet de limiter la consommation de l'ensemble des conteneurs d'un namespace.

exemple:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-quota
spec:
  hard:
    requests.memory: 1Gi
    limits.memory: 2Gi
```

Nous nous limiterons pour le moment à étudier les `limites` pour comprendre notre erreur 137. Mais il est tout aussi important de se pencher sur les `request` qui sont la base du système d'élection des noeuds et donc tout aussi sensible.

Regardons donc ce qu'il se passe à la création et le résultat dans docker sur le node.

```
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo
spec:
  containers:
  - name: default-mem-demo-ctr
    image: nginx
```

Si on regarde sur le node directement au travers de docker.

```
$ docker stats
CONTAINER ID        NAME                                                                                                          CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
62d6fe89f68e        k8s_default-mem-demo-ctr_default-mem-demo_default-mem-example_4571c6e9-55d5-11e9-a451-080027ac049f_0                0.00%               2.148MiB / 512MiB     0.42%               1.12kB / 0B         0B / 0B
        2
```

On voit bien que notre limite a bien été positionnée sur le conteneur.

Il est temps de refaire notre stress test.

```
apiVersion: v1
kind: Pod
metadata:
  name: stress-test
spec:
  containers:
  - name: membomb
    image: monitoringartist/docker-killer
    args: ["membomb"]
```

Maintenant regardons l'état de notre POD.

```
$ kubectl get pod stress-test -n default-mem-example
NAME          READY   STATUS      RESTARTS   AGE
stress-test   0/1     OOMKilled   2          28s
```

Regardons encore plus en détail
```
$ kubectl get pod stress-test --output=yaml
[...]
  containerStatuses:
  - containerID: docker://4ea051753d88a4d1e2af858b2dbdfa4400cdb13afbeea844109df885a1fc8561
    image: monitoringartist/docker-killer:latest
    imageID: docker-pullable://monitoringartist/docker-killer@sha256:85ba7f17a5ef691eb4a3dff7fdab406369085c6ee6e74dc4527db9fe9e448fa1
    lastState:
      terminated:
        containerID: docker://4ea051753d88a4d1e2af858b2dbdfa4400cdb13afbeea844109df885a1fc8561
        exitCode: 137
        finishedAt: "2019-04-03T06:41:24Z"
        reason: OOMKilled
        startedAt: "2019-04-03T06:41:24Z"
[...]
```

Et nous voilà à nouveau en face de notre fameuse erreur 137.

## Test CPU

Le raisonnement pour ces lmites CPU va être exactement le même que précédement pour la mémoire.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 0.8
    defaultRequest:
      cpu: 0.5
    max:
      cpu: 1
    min:
      cpu: 0.2
    type: Container
```

<u>remarque:</u><br>
Les limites peuvent aussi s'exprimer en millième ce qui donnerai pour 0.2 = 200m soit 2%

De la même façon que la mémoire, on peut agir sur des quota.

On peut aussi définir des quotas, ce qui permet de limiter la consommation de l'ensemble des conteneurs d'un namespace.

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-quota
spec:
  hard:
    requests.cpu: "1"
    limits.cpu: "2"
```

Nous pouvons donc reprendre notre petit stress test et voir comment notre conteneur se comporte.

```
apiVersion: v1
kind: Pod
metadata:
  name: stress-test
spec:
  containers:
  - name: membomb
    image: monitoringartist/docker-killer
    args: ["cpubomb"]
```

Regardons sur le node directement au travers de docker.

```
$ docker stats
CONTAINER ID        NAME                                                                                                          CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
d538c1c458da        k8s_cpubomb_stress-test_default-mem-example_fd6a8c28-55de-11e9-a451-080027ac049f_2                            81.24%              2.188MiB / 512MiB     0.43%               1.33kB / 0B         0B / 0B
```

On voit que notre conteneur est capé aux alentours de 80% qui est bien notre limite par défaut.

## Test disque

### Filesystem

Kubernetes introduit la notion de ephemeral-storage qui constitue la zone de sotckage allouée à un POD.
Cet espace de stockage représente la somme du stockage consommé (layer writable, logs, emptyDir volumes)

Comme les autres ressources il est possible d'y appliquer des limites.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: ephemeral-limit-range
spec:
  limits:
  - default:
      ephemeral-storage: "512Mi"
    defaultRequest:
      ephemeral-storage: "256Mi"
    type: Container
```

Dans l'exmple on applique des valeurs par défaut sur le namespace, mais on peut bien evidement les surcharger à la création de chacun des POD.

Je créé donc mon POD en ayant préalablement positionné les limites sur le namespace.

```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  containers:
  - name: linux
    image: ubuntu
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
```

je regarde l'espace disque alloué dans le POD

```
# kubectl exec -it ubuntu /bin/bash
root@ubuntu:/#  df -h

Filesystem      Size  Used Avail Use% Mounted on
overlay          17G  2.2G   13G  15% /
```
On peut voir que le disque n'indique pas la limite de ephemeral-storage mais l'espace de stockage du système hôte.
Il va donc falloir faire attention car il n'est pas possible dans les conteneurs de connaitre dynamiquement la limite définie. On risque donc de la dépasser facilement.

C'est ce qu'on va tester en créant un fichier fichier de 1G.

```
# kubectl exec -it ubuntu /bin/bash
root@ubuntu:/#  fallocate -l 1G test.img
```
Le fichier est bien créé et le POD n'est pas arrêté immediatement, mais si on attend un peu.

```
root@ubuntu:/# command terminated with exit code 137
```

Revoilà notre erreur 137

Si on regarde le description de notre POD
```
$ kubectl describe pod ubuntu
[...]
  Warning  Evicted           76s                    kubelet, minikube  Pod ephemeral local storage usage exceeds the total limit of containers 512Mi.
  Normal   Killing           76s                    kubelet, minikube  Killing container with id docker://linux:Need to kill Pod
```

Le POD a bien été errêté car il a dépassé les limites définies.

### Log

Par défaut les logs sont consultables par la commande `kubectl logs`.
C'est simple ça nous masque les noeuds et en plus on peut voir les logs d'un POD et plus précisement d'un conteneur.
On peut même voir avec l'option `--previous`, les logs d'un conteneur précédement tombé, sauf dans un seul cas : L'eviction
Si on revient à notre petite histoire. c'est justement notre cas. Donc on ne peut pas s'appuyer sur cette fonctionnalité pour comprendre notre problème.

Si on regarde du coté de la rotation. Kubernetes n'implémente rien nativement, il faut intervenir directement sur les conteneurs , la daemon des conteneurs ou un outil externe. Il faut aussi penser que la commande `kubelet logs` ne connait que le dernier fichier de rotation. Ca peut aider lorsqu'on ne trouve pas de trace, ce qui peut être normal durant à la phase de rotation.

Le meilleur moyen bien evidement reste l'implémentation d'un daemonset de type fluentd qui va s'occuper de la collecte, la rotation et la redirection des logs vers un outil externe (elasticsearch, graylog, ...)

[< Précédent](error-137-3.md) | [Sommaire](error-137-0.md) | [Suivant >](error-137-5.md)