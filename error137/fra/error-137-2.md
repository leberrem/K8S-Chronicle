# Comprendre

La premier reflexe lors de cette erreur est d'aller sur notre chère ami google.

Les réponses sur le sujet sont très diverses.
 - Bug du runner
 - Problème de limite du système hôte
 - Problème de dépassement mémoire

Du coup on tente de modifier les paramètres du runner par patch de la configmap.

Voici un exemple de configuration
```
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_CPU_LIMIT=500m
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_MEMORY_LIMIT=4Gi
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_CPU_REQUEST=250m
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_MEMORY_REQUEST=1Gi
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_CPU_LIMIT=250m
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_MEMORY_LIMIT=1.5Gi
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_CPU_REQUEST=100m
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_MEMORY_REQUEST=250Mi
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_CPU_LIMIT=250m
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_MEMORY_LIMIT=1Gi
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_CPU_REQUEST=100m
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_MEMORY_REQUEST=500Mi
```

Mais rien n'y fait c'est toujours instable

Du coup on revient en arrière

```
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_CPU_LIMIT-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_MEMORY_LIMIT-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_CPU_REQUEST-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_MEMORY_REQUEST-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_CPU_LIMIT-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_MEMORY_LIMIT-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_CPU_REQUEST-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_SERVICE_MEMORY_REQUEST-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_CPU_LIMIT-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_MEMORY_LIMIT-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_CPU_REQUEST-
kubectl set env deploy/runner-gitlab-runner -n gitlab-managed-apps KUBERNETES_HELPER_MEMORY_REQUEST-
```
On regarde chacun des noeuds mais tout se passe bien la CPU ronronne et la mémoire est tranquille.

On tente de trouver des traces par commande `kubectl describe` mais malhereusement les PODs sont détruit et ne laissent que peut de logs.

Il est temps d'arrêter de tatonner et de reprendre à la base.

Quelles informations nous donnent les logs de la CI/CD?

```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on runner-gitlab-runner-76d846db5b-sk4xd hi4Fn6Ay
Using Kubernetes namespace: gitlab-managed-apps
Using Kubernetes executor with image openjdk:11-jdk-slim ...
Waiting for pod gitlab-managed-apps/runner-hi4fn6ay-project-28-concurrent-0khrql to be running, status is Pending
Running on runner-hi4fn6ay-project-28-concurrent-0khrql via runner-gitlab-runner-76d846db5b-sk4xd...
Cloning repository...
Cloning into '/sabhd/dev-hat/consent'...
Checking out 8f156473 as master...
Skipping Git submodules setup
Checking cache for master...
Downloading cache.zip from http://minio-service.gitlab-ci:9000/gitlab-cache/%20/runner/hi4Fn6Ay/project/28/master 
Successfully extracted cache
Downloading artifacts for build (4800)...
Downloading artifacts from coordinator... ok        id=4800 responseStatus=200 OK token=c4f-J285
[...]
.
.
.
[...]
ERROR: job failed: command terminated with exit code 137
```

On y retrouve la version du runner : `gitlab-runner 11.8.0`<br>
Le pod correspondant : `runner-gitlab-runner-76d846db5b-sk4xd`<br>
Et plus interessant le pod du traitement : `runner-hi4fn6ay-project-28-concurrent-0khrql`<br>

A partir de ces informations on va pouvoir regarder sur chacun des noeuds ce qui se passe sur le journal des services.

On commence par regarder sur chacun des noeuds qui parle de notre conteneur.

`journalctl | grep runner-hi4fn6ay-project-28-concurrent-0khrql `

```
Mar 22 15:26:57 node3 kubelet[7481]: I0322 15:26:57.027454    7481 kubelet.go:1883] SyncLoop (ADD, "api"): "runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62)"
[...]
.
.
.
[...]
Mar 22 15:28:11 node3 kubelet[7481]: I0322 15:28:11.663393    7481 eviction_manager.go:563] eviction manager: pod runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62) is evicted successfully
Mar 22 15:28:11 node3 kubelet[7481]: I0322 15:28:11.663452    7481 eviction_manager.go:187] eviction manager: pods runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62) evicted, waiting for pod to be cleaned up
Mar 22 15:28:41 node3 kubelet[7481]: W0322 15:28:41.663651    7481 eviction_manager.go:392] eviction manager: timed out waiting for pods runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62) to be cleaned up

```
Du coup on peut voir que le conteneur est rentré dans un processus d'éviction et que celui-ci a été supprimé.
J'ai donc l'explication de mon erreur 137.

Pour autant pourquoi mon POD est-il rentré? dans un processus d'eviction?

Si on regarde plus précisement `kubelet` et particulièrement `eviction_manager`.

`journalctl -u kubelet | grep eviction_manager`

```
Mar 22 10:53:39 node3 kubelet[7481]: E0322 10:53:39.307356    7481 eviction_manager.go:243] eviction manager: failed to get get summary stats: failed to get node info: node "node3" not found
Mar 22 10:58:40 node3 kubelet[7481]: W0322 10:58:40.106059    7481 eviction_manager.go:329] eviction manager: attempting to reclaim ephemeral-storage
Mar 22 10:58:41 node3 kubelet[7481]: I0322 10:58:41.752662    7481 eviction_manager.go:336] eviction manager: able to reduce ephemeral-storage pressure without evicting pods.
Mar 22 15:20:17 node3 kubelet[7481]: W0322 15:20:17.167069    7481 eviction_manager.go:329] eviction manager: attempting to reclaim ephemeral-storage
Mar 22 15:20:25 node3 kubelet[7481]: I0322 15:20:25.774164    7481 eviction_manager.go:336] eviction manager: able to reduce ephemeral-storage pressure without evicting pods.
Mar 22 15:27:27 node3 kubelet[7481]: W0322 15:27:27.005912    7481 eviction_manager.go:329] eviction manager: attempting to reclaim ephemeral-storage
Mar 22 15:27:33 node3 kubelet[7481]: I0322 15:27:33.357105    7481 eviction_manager.go:340] eviction manager: must evict pod(s) to reclaim ephemeral-storage
Mar 22 15:27:33 node3 kubelet[7481]: I0322 15:27:33.357616    7481 eviction_manager.go:358] eviction manager: pods ranked for eviction: rook-ceph-osd-2-759b67cc7c-62r7c_rook-ceph(0da13480-4c71-11e9-921d-0a341618daf2), mongodb-primary-0_mongodb(e838c138-4c90-11e9-921d-0a341618daf2), rook-ceph-mon-c-6945d7f6db-mmmbr_rook-ceph(0d368597-4c71-11e9-921d-0a341618daf2), runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62), runner-hi4fn6ay-project-54-concurrent-04kpdn_gitlab-managed-apps(e8838f63-4cb6-11e9-a5e8-02d46899ed62), rook-discover-2m58t_rook-ceph-system(2306e30d-305e-11e9-921d-0a341618daf2), rook-ceph-agent-z6p7s_rook-ceph-system(22fcc389-305e-11e9-921d-0a341618daf2), nginx-proxy-node3_kube-system(f1188d90a8fa1b4b2b364a6c69a5deba), calico-node-vw4nq_kube-system(d7dedece-2e46-11e9-bfea-0a341618daf2), kube-proxy-rcnvh_kube-system(f05935fe-2e46-11e9-bfea-0a341618daf2)
Mar 22 15:27:33 node3 kubelet[7481]: W0322 15:27:33.420424    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-7t94k_rook-ceph(038e4ad3-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:33 node3 kubelet[7481]: W0322 15:27:33.461872    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-vblsb_rook-ceph(03968b87-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:33 node3 kubelet[7481]: W0322 15:27:33.505508    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-8v767_rook-ceph(039c7e80-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:33 node3 kubelet[7481]: W0322 15:27:33.542242    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-8hl8c_rook-ceph(03a34d3c-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:33 node3 kubelet[7481]: W0322 15:27:33.588833    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-r52gj_rook-ceph(03a9c32e-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:33 node3 kubelet[7481]: W0322 15:27:33.788281    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-tjt8c_rook-ceph(03c849fe-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:34 node3 kubelet[7481]: W0322 15:27:34.193436    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-d5kns_rook-ceph(04062268-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:34 node3 kubelet[7481]: W0322 15:27:34.607833    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-scwdc_rook-ceph(04423088-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:34 node3 kubelet[7481]: I0322 15:27:34.717434    7481 eviction_manager.go:563] eviction manager: pod rook-ceph-osd-2-759b67cc7c-62r7c_rook-ceph(0da13480-4c71-11e9-921d-0a341618daf2) is evicted successfully
Mar 22 15:27:34 node3 kubelet[7481]: I0322 15:27:34.717466    7481 eviction_manager.go:187] eviction manager: pods rook-ceph-osd-2-759b67cc7c-62r7c_rook-ceph(0da13480-4c71-11e9-921d-0a341618daf2) evicted, waiting for pod to be cleaned up
Mar 22 15:27:34 node3 kubelet[7481]: W0322 15:27:34.986713    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-l8t8m_rook-ceph(047f1801-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:35 node3 kubelet[7481]: W0322 15:27:35.390993    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-75ml7_rook-ceph(04bc6e9d-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:35 node3 kubelet[7481]: W0322 15:27:35.787492    7481 eviction_manager.go:156] Failed to admit pod rook-ceph-osd-2-759b67cc7c-w8p4w_rook-ceph(04f9a19c-4cb7-11e9-921d-0a341618daf2) - node has conditions: [DiskPressure]
Mar 22 15:27:37 node3 kubelet[7481]: I0322 15:27:37.722573    7481 eviction_manager.go:400] eviction manager: pods rook-ceph-osd-2-759b67cc7c-62r7c_rook-ceph(0da13480-4c71-11e9-921d-0a341618daf2) successfully cleaned up
Mar 22 15:27:37 node3 kubelet[7481]: W0322 15:27:37.737276    7481 eviction_manager.go:329] eviction manager: attempting to reclaim ephemeral-storage
Mar 22 15:27:37 node3 kubelet[7481]: I0322 15:27:37.853406    7481 eviction_manager.go:340] eviction manager: must evict pod(s) to reclaim ephemeral-storage
Mar 22 15:27:37 node3 kubelet[7481]: I0322 15:27:37.853853    7481 eviction_manager.go:358] eviction manager: pods ranked for eviction: mongodb-primary-0_mongodb(e838c138-4c90-11e9-921d-0a341618daf2), rook-ceph-mon-c-6945d7f6db-mmmbr_rook-ceph(0d368597-4c71-11e9-921d-0a341618daf2), runner-hi4fn6ay-project-54-concurrent-04kpdn_gitlab-managed-apps(e8838f63-4cb6-11e9-a5e8-02d46899ed62), runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62), rook-discover-2m58t_rook-ceph-system(2306e30d-305e-11e9-921d-0a341618daf2), rook-ceph-agent-z6p7s_rook-ceph-system(22fcc389-305e-11e9-921d-0a341618daf2), nginx-proxy-node3_kube-system(f1188d90a8fa1b4b2b364a6c69a5deba), calico-node-vw4nq_kube-system(d7dedece-2e46-11e9-bfea-0a341618daf2), kube-proxy-rcnvh_kube-system(f05935fe-2e46-11e9-bfea-0a341618daf2)
Mar 22 15:27:39 node3 kubelet[7481]: I0322 15:27:39.461326    7481 eviction_manager.go:563] eviction manager: pod mongodb-primary-0_mongodb(e838c138-4c90-11e9-921d-0a341618daf2) is evicted successfully
Mar 22 15:27:39 node3 kubelet[7481]: I0322 15:27:39.461362    7481 eviction_manager.go:187] eviction manager: pods mongodb-primary-0_mongodb(e838c138-4c90-11e9-921d-0a341618daf2) evicted, waiting for pod to be cleaned up
Mar 22 15:28:09 node3 kubelet[7481]: W0322 15:28:09.461599    7481 eviction_manager.go:392] eviction manager: timed out waiting for pods mongodb-primary-0_mongodb(e838c138-4c90-11e9-921d-0a341618daf2) to be cleaned up
Mar 22 15:28:09 node3 kubelet[7481]: W0322 15:28:09.493120    7481 eviction_manager.go:329] eviction manager: attempting to reclaim ephemeral-storage
Mar 22 15:28:10 node3 kubelet[7481]: I0322 15:28:10.391610    7481 eviction_manager.go:340] eviction manager: must evict pod(s) to reclaim ephemeral-storage
Mar 22 15:28:10 node3 kubelet[7481]: I0322 15:28:10.391935    7481 eviction_manager.go:358] eviction manager: pods ranked for eviction: runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62), runner-hi4fn6ay-project-54-concurrent-04kpdn_gitlab-managed-apps(e8838f63-4cb6-11e9-a5e8-02d46899ed62), rook-ceph-mon-c-6945d7f6db-mmmbr_rook-ceph(0d368597-4c71-11e9-921d-0a341618daf2), rook-discover-2m58t_rook-ceph-system(2306e30d-305e-11e9-921d-0a341618daf2), rook-ceph-agent-z6p7s_rook-ceph-system(22fcc389-305e-11e9-921d-0a341618daf2), nginx-proxy-node3_kube-system(f1188d90a8fa1b4b2b364a6c69a5deba), calico-node-vw4nq_kube-system(d7dedece-2e46-11e9-bfea-0a341618daf2), kube-proxy-rcnvh_kube-system(f05935fe-2e46-11e9-bfea-0a341618daf2)
Mar 22 15:28:11 node3 kubelet[7481]: I0322 15:28:11.663393    7481 eviction_manager.go:563] eviction manager: pod runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62) is evicted successfully
Mar 22 15:28:11 node3 kubelet[7481]: I0322 15:28:11.663452    7481 eviction_manager.go:187] eviction manager: pods runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62) evicted, waiting for pod to be cleaned up
Mar 22 15:28:41 node3 kubelet[7481]: W0322 15:28:41.663651    7481 eviction_manager.go:392] eviction manager: timed out waiting for pods runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62) to be cleaned up

```

Ce qu'on peut voir c'est que le processus d'éviction a été lancé pour une demande d'espace de stockage
`eviction manager: must evict pod(s) to reclaim ephemeral-storage`

Du coup je vais avoir un problème d'espace disque à gérer sur mes noeuds.

Une autre chose me choque

`eviction manager: pods ranked for eviction: runner-hi4fn6ay-project-28-concurrent-0khrql_gitlab-managed-apps(edde7e59-4cb6-11e9-a5e8-02d46899ed62), runner-hi4fn6ay-project-54-concurrent-04kpdn_gitlab-managed-apps(e8838f63-4cb6-11e9-a5e8-02d46899ed62), rook-ceph-mon-c-6945d7f6db-mmmbr_rook-ceph(0d368597-4c71-11e9-921d-0a341618daf2), rook-discover-2m58t_rook-ceph-system(2306e30d-305e-11e9-921d-0a341618daf2), rook-ceph-agent-z6p7s_rook-ceph-system(22fcc389-305e-11e9-921d-0a341618daf2), nginx-proxy-node3_kube-system(f1188d90a8fa1b4b2b364a6c69a5deba), calico-node-vw4nq_kube-system(d7dedece-2e46-11e9-bfea-0a341618daf2), kube-proxy-rcnvh_kube-system(f05935fe-2e46-11e9-bfea-0a341618daf2)`

Il y a tout un tas de POD qui rentrent dans le processus d'eviction qui sont pour moi particulièrement sensibles.
pour exemple : `rook-ceph`, `kube-proxy`, `calico-node`




[< Précédent](error-137-1.md) | [Sommaire](error-137-0.md) | [Suivant >](error-137-3.md)