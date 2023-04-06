# Controllers

## Replica Set (ou Replication controller - legacy)

> Utile pour avoir une HA, pour du LB et du Scaling. 

> Il peut agir au dela du scope d'un node worker

![replication_controller](./images/replication_controller.png)

Dans le def file, `template` inclut un Pod definition file.

Pour les replicasets, c'est comme les replica controllers mais l'idée c'est de décorreler la conf avec celle des pods via la clé `selectors` en utilisant `matchLabels` p ex

Question : à quelle moment est utilisé la clé `template` ? est ce que ca veut dire que les replicas peuvent différer de la definition du pod ? quel interet ?

<details>
à noter que lorsque l'on modifie le template des pods supervisés par un replicaset, il est parfois nécessaire de tuer les pods afin que la modification soit appliquée
</details>


## Deployment

> C'est la partie qui nous permet de configurer comment vont se déployer nos pods.

RQ : la conf Deployment "encapsule" la conf Replica Sets

![Deployment](./images/deployment.png)

# Namespace

> C'est un scope pour les ressources kube (services + objets - i.e.pods&controllers) utilisé pour de l'**isolation**

À noter que Kube crée 3 namespaces par défaut :
* `Default` : à ne pas utiliser en production
* `kube-system` : tous les objets utile à Kube lui meme pour faire fonctionner le cluster
* `kube-public` : pour les objets qui devraient être accessibles pour tout le monde

À noter qu'une entrée DNS est attribué par défaut a tous les pods et service grace à kube-dns :
* dans le meme namespace, il suffit d'appeler le nom du service (p. ex. `mysql.connect(<service-name>)` ou bien HTTP request sur le host <pod-name>) 
* dans 2 namespaces différents, `<service-name>.<namespace>.svc.cluster.local`

<details><summary>CLI ex</summary>

Pour la création de resource kube, on peut soit spécifier `--namespace=` dans la command kubectl soit utiliser la clé `namespace:` dans le def file sous le noeud `metadata:`

On peut créer des namespaces via un def file ou via la command `kubectl create`
</details>

<details><summary>Pratique : environnements multiples & limites de ressources</summary>
Les namespaces ont été prévu pour les environnements dev/preprod/staging/qa/prod car on peut allouer des limites de resources par namespace via l'objet `ResourceQuota`. Mais en pratique c'est encore mieux isolé d'utiliser des clusters différents.
</details>

# Services

> C'est une couche abstraite pour configurer la liaison vers les Pods. 

## Cluster IP

> Utiles pour allouer une IP interne virtuelle à une ressource kube e.g. un pod

Il s'appuie lui aussi sur les labels d'un pod à travers la clé `selector`

![cluster ip](./images/cluster_ip_def_file.png)

## Node Port

> Expose un (groupe de) pod sur un port donné

![ex d'usage d'un service](./images/ex_services_usage.png)

<details>
  <summary>Attention : **Cela n'ouvre pas un accès depuis l'extérieur**</summary>, pour l'instant il est quand meme nécessaire de se trouver dans le même réseau que celui du cluster kube. 
</details>

--

Il est nécessaire de préciser 3 ports pour cette conf : 
![node port def file](./images/service_type_nodeport_def_file.png)
![node port with selector](./images/service_type_nodeport_w_selector.png)

À noter que la clé `ports` est au pluriel et attend donc une liste. 

<details>
De la même maniere que pour les replica set, le service nodeport prévoit un clé `selector` de manière à identifier les pods qu'il doit servir via les labels du pod. Ainsi, tous les pods, même s'ils tournent sur différent `worker nodes`, seront servis par le service NodePort. L'algo pour choisir au "runtime" est `random`. Ainsi, il existe _out-of-the-box_ un `load balancing` dans le Node Port kube (cf. screen ci dessous)
</details>

![service_nodeport_w_multiple_node_workers](./images/service_nodeport_w_multiple_node_workers.png)

À noter que le service est dispo dans tous les node workers du cluster. Ainsi, meme si par malheur le node que l'on choisi n'héberge pas le pod désiré, il répondra quand meme correctement comme tous les autres nodes workers

## LoadBalancer

> Si le cloud provider le supporte, il est possible d'utiliser cette conf pour mapper un nom de domaine pour un groupe de pod

On utilise la meme conf que le node port mais en créant un service type `LoadBalancer` à la place.
