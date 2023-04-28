# Storage

## Docker Storage

### Storage drivers

Docker stocke toute sa data (image, containers, etc) dansle dir `/var/lib/docker`

Layered architecture = chaque ligne d'un dockerfile est une layer et Docker réutilise les layers identiques au moment du `docker build` meme entre differents dockerfiles. À noter qu'à ce niveau il s'agit de layer read only. Le seul moyen de les editer est de relancer un `docker build`

Quand on lance un `docker run`, docker génère alos une **writable** layer car elle permet au container d'écrire dedans (logs, temp files, etc) ou meme quand le user écrit directement dedans. De la meme maniere, cette nouvelle layer peut etre utilisée par d'autres. Lorsque le container est destroy, la data est supprimée.

Docker choisit le storage driver (AUFS, ZFS, Overlay, Overlay2, etc) en fonction de l'OS

#### COPY ON WRITE de Docker

Remarque : si la code base a été copiée dans une layer read only au build de l'image, lorsque l'on run le container ET que l'on edite la code base alors juste avant la sauvegarde docker créera une copie des fichiers modifiés dans la couche writable.

### Volume drivers

C'est le systeme de Persistence de donnée de Docker de maniere à conserver les modifs meme lorsque le container est destroy

Par defaut, il stocke la donnée dans `/var/lib/docker/volumes`
On peut lui dire où écrire auquel cas on fait du volume binding

Dans le cas des volumes, on ne parle pas de storage driver. C'est le user qui choisit le driver (ou plugin) et il en existe plein pour stocker dans le cloud p ex.

## Kubernetes

### Container Runtime Interface CRI

K8s a créé une interface pour ne pas être "vendor lock" avec docker et pour que d'autres solutions de container runtime puisse etre utilisé avec Kubernetes. De la meme maniere, `CNI` est l'interface de kube pour le networking, et `CSI` pour le storage.

### Volumes

Pour utiliser un volume avec Kubernetes, cela se fait en 2 étapes :
1. création du volume dans la solution de Storage
  * Dans un def file, on utilise le noeud `volumes:` (à la même hauteur que `containers:`)
2. montage du volume avec le pod
  * Dans un def file on utilise le noeud `volumeMounts:` pour chaque container du pod

Il existe plusieurs `storage type solutions` (GCP, AWS, etc) et il est meme possible d'écrire sur le host cad le node (attention car la donnée ne sera pas les memes sur tous les nodes du coup ^^) 

#### PVC

C'est une manière de centraliser la configuration du system de persistence utilisé dans un cluster. 

On le crée avec un def file `kind: PersistenceVolume`

Une fois qu'un PV est configuré, on peut le binder à un pod au travers de ce qui s'appelle un PVC (persistent volume claim). La relation PV-PVC est oneToOne donc meme si tout l'espace du PV n'est pas utilisé par le PVC alors il n'est pas possible de binder l'espace restant à un autre PVC. S'il n'y a plus de PV dispo, un PVC peut rester en status PENDING.

On crée un PVC via un def file `kind: PersistentVolumeClaim`.

<details>K8s choisit le meilleur PV en fonction de l'espace disponible, du AccessMode (~ les permissions souhaitées e.g. readonly, readandwrite) etc, mais il est aussi possible d'utiliser des labels et selectors pour choisir nous meme.
Lorsque le PVC est détruit, par def k8s va Retain la donnée dans le volume et donc le PV ne sera toujours pas dispo pour d'autres PVC. On peut changer ça pour que le PV soit Delete ou bien Recycle</details>

On peut enfin utiliser un PVC dans un pod en tant que volume. 

#### Dynamic provisioning: Storage Class

Pour éviter d'avoir à provisioner (créer) les espaces de stockage à chaque fois un PV est créé, on peut utiliser un `kind: StorageClass`
