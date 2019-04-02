# Retour aux sources

Ce que je propose pour bien comprendre c'est de repartir sur les bases de docker et de vérifier les limites et tenter de voir le comportement des conteneurs. On transposera ensuite cette logique dans Kubernetes.

## Test mémoire

Si vous voulez connaitre l'espace mémoire disponible dans un conteneur

```
$ docker run --memory 512m --rm -it ubuntu bash
root@73b135528456:/# cat /sys/fs/cgroup/memory/memory.limit_in_bytes
536870912
```

Ok tout va bien la limite est positionnée et respectée.

Et si on tente de pousser le système

```
$ docker run --rm -ti --memory 512M monitoringartist/docker-killer membomb
membomb - duration 60s
Test: excessive memory (RAM+swap) utilization
Mem: 4532320K used, 3635984K free, 305052K shrd, 180852K buff, 2356808K cached
Mem: 4937580K used, 3230724K free, 304932K shrd, 180852K buff, 2356772K cached
Mem: 4938380K used, 3229924K free, 304936K shrd, 180852K buff, 2356716K cached
Mem: 4938124K used, 3230180K free, 304940K shrd, 180852K buff, 2356620K cached
Mem: 4936960K used, 3231344K free, 304816K shrd, 180852K buff, 2356596K cached
Mem: 4940012K used, 3228292K free, 304952K shrd, 180852K buff, 2356916K cached
Mem: 4937220K used, 3231084K free, 304956K shrd, 180860K buff, 2356736K cached
environment: line 1:    11 Killed                  /membomb
$ echo $?
137
```

Notre conteneur a bien été arrêté et on retrouve notre erreur 137

Si on regarde les évènements docker, on retrouve un évènement `container oom` (Out of Memory Management).
```
$ docker system events
2019-04-02T11:07:40.741590832+02:00 container create 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (image=monitoringartist/docker-killer, name=cranky_payne)
2019-04-02T11:07:40.743023953+02:00 container attach 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (image=monitoringartist/docker-killer, name=cranky_payne)
2019-04-02T11:07:40.838466953+02:00 network connect 3fa825bda4a43d4c75ddc697877291417fa0ea1ac4ff7d618fe9392a381ae8c4 (container=779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c, name=bridge, type=bridge)
2019-04-02T11:07:41.602707930+02:00 container start 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (image=monitoringartist/docker-killer, name=cranky_payne)
2019-04-02T11:07:41.606932429+02:00 container resize 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (height=24, image=monitoringartist/docker-killer, name=cranky_payne, width=182)
2019-04-02T11:07:45.230303858+02:00 container oom 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (image=monitoringartist/docker-killer, name=cranky_payne)
2019-04-02T11:07:45.565069706+02:00 container die 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (exitCode=137, image=monitoringartist/docker-killer, name=cranky_payne)
2019-04-02T11:07:45.678982268+02:00 network disconnect 3fa825bda4a43d4c75ddc697877291417fa0ea1ac4ff7d618fe9392a381ae8c4 (container=779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c, name=bridge, type=bridge)
2019-04-02T11:07:45.757197862+02:00 container destroy 779ea5a6dcf0538c93941f2e9574bd05e07b7c2a545bb1598e42d63be68c677c (image=monitoringartist/docker-killer, name=cranky_payne)
```

Par contre si on ne positionne aucune limite, voici ce qu'il se passe.

```
docker run --rm -it ubuntu bash
root@66c71e1e685e:/# cat /sys/fs/cgroup/memory/memory.limit_in_bytes
9223372036854771712
```

La valeur remontée est bien supérieure à ce que peut fournir le système.
Par contre la commande `docker stats` indique bien la mémoire disponible du système hôte.

```
ONTAINER ID        NAME                 CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
9f08b32cf82a        flamboyant_davinci   0.00%               1.34MiB / 7.79GiB   0.02%               2.59kB / 0B         8.86MB / 0B         1
```

On se lance et on execute un petit test.

```
docker run --rm -ti  monitoringartist/docker-killer membomb
membomb - duration 60s
Test: excessive memory (RAM+swap) utilization
Mem: 4967776K used, 3200528K free, 315404K shrd, 182080K buff, 2371444K cached
Mem: 5540048K used, 2628256K free, 319280K shrd, 182080K buff, 2375352K cached
Mem: 6089552K used, 2078752K free, 319256K shrd, 182080K buff, 2375328K cached
Mem: 6640760K used, 1527544K free, 312692K shrd, 182080K buff, 2368764K cached
Mem: 7219172K used, 949132K free, 315456K shrd, 182080K buff, 2371528K cached
Mem: 7782580K used, 385724K free, 315808K shrd, 182080K buff, 2371880K cached
Mem: 8046140K used, 122164K free, 315692K shrd, 177620K buff, 1929544K cached
Mem: 8044092K used, 124212K free, 312028K shrd, 155180K buff, 949816K cached
Mem: 8063056K used, 105248K free, 305800K shrd, 15220K buff, 526268K cached
Mem: 8061016K used, 107288K free, 296948K shrd, 1288K buff, 329672K cached
Mem: 8062988K used, 105316K free, 295800K shrd, 824K buff, 315792K cached
Mem: 8066072K used, 102232K free, 295780K shrd, 360K buff, 316888K cached
Mem: 8065884K used, 102420K free, 295288K shrd, 596K buff, 317984K cached
Mem: 8067680K used, 100624K free, 291712K shrd, 1560K buff, 315200K cached
Mem: 8067504K used, 100800K free, 291692K shrd, 144K buff, 292780K cached
environment: line 1:    11 Killed                  /membomb
leberrem@elementary:~/workspace/github/K8S-Chronicle$ echo $?
137
```
On voit que la valeur monte jusqu'à saturation du système hote (limité à 8GB pour mon cas)
Du coup même chose que lors du précédent test avec lmite mémoire, on regarde les evènements docker.

```
$ docker system events
2019-04-02T11:10:08.929492701+02:00 container create 7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4 (image=monitoringartist/docker-killer, name=pedantic_golick)
2019-04-02T11:10:08.930417120+02:00 container attach 7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4 (image=monitoringartist/docker-killer, name=pedantic_golick)
2019-04-02T11:10:08.982912967+02:00 network connect 3fa825bda4a43d4c75ddc697877291417fa0ea1ac4ff7d618fe9392a381ae8c4 (container=7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4, name=bridge, type=bridge)
2019-04-02T11:10:09.692052000+02:00 container start 7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4 (image=monitoringartist/docker-killer, name=pedantic_golick)
2019-04-02T11:10:09.698323246+02:00 container resize 7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4 (height=24, image=monitoringartist/docker-killer, name=pedantic_golick, width=182)
2019-04-02T11:10:25.352241745+02:00 container die 7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4 (exitCode=137, image=monitoringartist/docker-killer, name=pedantic_golick)
2019-04-02T11:10:25.902968674+02:00 network disconnect 3fa825bda4a43d4c75ddc697877291417fa0ea1ac4ff7d618fe9392a381ae8c4 (container=7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4, name=bridge, type=bridge)
2019-04-02T11:10:26.663866591+02:00 container destroy 7721ec3bd28ef8da4e7a46e8192adc738a20c1a489f9f7c6f3ea478220dc09e4 (image=monitoringartist/docker-killer, name=pedantic_golick)
```

Petit différence car on ne retrouve plus l'évènement `container oom`.
C'est le processus standard OOM du système qui prend le relais pour éviter la saturation de notre serveur.
Il faut bien prendre conscience que dans ce cas c'est le bien système qui choisi quoi arrêter et peut selectionner d'autres processus plus critiques que notre conteneur. Cependant docker ne défini pas de priorité OOM sur les conteneurs, mais en positionne une sur le daemon docker. de fait les conteneurs seront "killé" avant le daemon parent. Ouf...

## Test CPU

Maintenant qu'on a vu le comportement sur la mémoire. voyons ce qu'il se passe sur le CPU.

```
$ docker run --rm -ti --cpus=".5" monitoringartist/docker-killer cpubomb
cpubomb - duration 60s
Test: excessive CPU utilization - one proces per processor with empty cycles
CPU:  84% usr  11% sys   0% nic   3% idle   0% io   0% irq   0% sirq
CPU:  61% usr   4% sys   0% nic  34% idle   0% io   0% irq   0% sirq
CPU:  79% usr  10% sys   0% nic   9% idle   0% io   0% irq   0% sirq
CPU:  54% usr   5% sys   0% nic  40% idle   0% io   0% irq   0% sirq
CPU:  51% usr   3% sys   0% nic  45% idle   0% io   0% irq   0% sirq
CPU:  71% usr   7% sys   0% nic  20% idle   0% io   0% irq   0% sirq
CPU:  65% usr   5% sys   0% nic  29% idle   0% io   0% irq   0% sirq
```

La commande `docker stats` montre que le conteneur est limité aux alentours de 50%.

```
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT    MEM %               NET I/O             BLOCK I/O           PIDS
4e39e35fe1eb        trusting_robinson   50.03%              2.438MiB / 7.79GiB   0.03%               2.69kB / 0B         2.02MB / 0B         9
```

On peut donc jouer sur cette limite pour répartir la CPU entre plusieurs conteneurs, mais on ne pourra jamais attribuer plus que la quantité disponible sur le système hôte. Moins de risque dans ce cas d'être "killé" par un processus système car la CPU restante sera automatiquement redistribuée.

Si vous voulez aller plus loin dans les valeurs CPU/MEM, vous pouvez consulter le doc officielle docker :<br>
https://docs.docker.com/config/containers/resource_constraints/

Un autre lien vers<br> la documentation du conteneur de stress test:<br>
https://hub.docker.com/r/monitoringartist/docker-killer/

## Test disque

### Filesystem

Avant tout il faut bien faire la différence entre un point de montage dans un conteneur qui vient du système hôte et l'espace disque éphémère qui sera utilisable et attribuable dans un conteneur.

Nous allons nous concentrer ici sur l'espace éphémère.

```
$ docker run --rm -it ubuntu bash
root@4628787a64af:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          20G   14G  4.6G  76% /
```

Le disque principal est représenté par `overlay`.
C'est tout simplement l'espace disque disponible sur le système hôte là où le dossier `Docker Root Dir` est positionné.
La plus part du temps ce dossier est dans `/var/lib/docker` mais si celui-ci a été déplacé on peut le retrouver par la commande `docker info`.

```
$ df -h
Sys. de fichiers Taille Utilisé Dispo Uti% Monté sur
/dev/sda1           20G     14G  4,6G  76% /
```


docker run -it --storage-opt size=120G fedora /bin/bash

Du coup il va falloir être vigilant avec cet espace de stockage.

Il est possible de le deplacer et d'en choisir l'emplacement.
Pour ce faire il faudra passer un nouvel argument au daemon docker `-g <folder>`
J'ai fait un petit role ansible pour automatiser tout ça : https://gitlab.com/leberrem/move-docker-filesystem


Pour la surveillance du dossier, il faudra travailler sur la purge régulière des images et conteneur.
docker propose quelques commandes de base.
exemples : 
* Suppression des volumes non utilisés : `docker volume rm $(docker volume ls -qf dangling=true)`
* Suppression des images, conteneurs, network, volumes non actifs/utilisés : `docker system prune -a -f --volumes`
* Suppression de l'ensemble des conteneurs : `docker rm -vf $(docker ps -aq)`
* Suppression de l'ensemble des images : `docker rmi -f $(docker images -aq)`

Pour aller plus loin (ou comme source d'inspiration) Spotify propose une image docker permettant de lancer quelques commandes de purge avec différents scénarios : https://github.com/spotify/docker-gc

### Log

Il ne faut pas non plus oublié un autre point de saturation possible qui sont les logs des conteneurs.
En effet les conteneurs pour respecter le `The Twelve-Factor App` tracent dans la STDOUT mais sur l'hote les traces sont stockées sur le filesystem.
`<Docker Root Dir>/containers/<ID container>/<ID container>-json.log`

Par dafaut il n'y a pas de rotation.

Il existe quelques options permettant de régler le comportement par défaut.
https://docs.docker.com/config/containers/logging/json-file/

Voici un petit test de rotation

```
$ docker run --rm --log-opt max-size=10k --log-opt max-file=3 ubuntu /bin/sh -c 'while true; do echo $(date); sleep 0.1; done'

$ ls -l /var/lib/docker/containers/535e254c0eb27a4d711efc6994e5e4878a63053b1c2f7be1832cedf66b82493b/*.log*
-rw-r----- 1 root root  3534 avril  2 15:40 535e254c0eb27a4d711efc6994e5e4878a63053b1c2f7be1832cedf66b82493b-json.log
-rw-r----- 1 root root 10096 avril  2 15:40 535e254c0eb27a4d711efc6994e5e4878a63053b1c2f7be1832cedf66b82493b-json.log.1
-rw-r----- 1 root root 10092 avril  2 15:40 535e254c0eb27a4d711efc6994e5e4878a63053b1c2f7be1832cedf66b82493b-json.log.2
```

<u>Remarque:</u><br>
J'arrêterai la démonstration ici, mais il est possible d'agir sur d'autres paramètres de limite comme les forks, le réseau, etc...


[< Précédent](error-137-2.md) | [Sommaire](error-137-0.md) | [Suivant >](error-137-4.md)