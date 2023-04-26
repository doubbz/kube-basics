# Scheduling

En gros il y a 4 phases dans le scheduling : 

1. Scheduling queue: dans le cas où plusieurs pods sont à scheduler, on va pouvoir les trier par priorité (cf. `kind: priorityClass`)
2. Filtering: ici on exclut les nodes qui vont pas du tout
3. Scoring: le fameux scoring pour choisir le meilleur node parmi ceux qui restent après le filtering
4. Binding: le pod va être bindé à un node

<details><summary>Customisation</summary>

À chaque phase, il existe un ou plusieurs `extensions points` pour customiser le scheduling de kube.

À savoir qu'il est d'ailleurs possible de créer des scheduler custom qui viennent s'ajouter ou remplacer le default scheduler. 

Important de savoir qu'il est possible de customiser le scheduling via l'objet `kind: KubeSchedulerConfiguration` et les `scheduler profiles` sans avoir à écrire 1 ligne de code
</details>

## Manual conf

> On force l'assignation sur un node (kube-scheduler ne sera pas utile) grace à la clé `nodeName` dans un def file

<details>

Si le pod est deja créé, on ne peut plus modifier la clé nodeName. Du coup on peut utiliser la meme methode que le kube-scheduler : créer un objet binding via une POST request sur `/pods/$PODNAME/binding` avec le def file `kind: Binding` JSONifié
</details>

## Labels & selectors

> On regroupe des resources via des labels pour les retrouver par la suite

On peut :
* mettre des labels sur les resources
* selectionner les ressources dans les services, replicasets ou les deployements

<details><summary>confusant : dans un replicaset on retrouve le meme label écrit 3x</summary>

Feature basique de kube mais il est intéressant de se concentrer sur une partie un peu plus confusante lorsque l'on crée une ressource qui en gère une autre comme un replicaset ou un deployment p ex. Ces derniers s'appuient sur les labels justement. On se retrouve avec le meme label écrit 3 fois dans le def file comme dans le screen ci-dessous

![labels and selectors](./images/labels%20and%20selectors.png)

* le 1er est utilisé pour labeliser le replicaset (le pratique on utilise souvent les memes labels que les pods mais en soi on peut mettre ce qu'on veut)
* le 2e sert pour déterminer les pods qui doivent etre gérés par le replicaset
* le 3e sert à labéliser les pods (il est courant d'avoir des labels qui ne sont pas forcément utilisé dans le select du replicaset)

</details>

## Taints and tolerations

> C'est une maniere d'éviter que certains node soient selectionnés par le scheduler kube dans le deploiement de PODs.

Penser à l'analogie du repulsif d'insecte e.g. anti-moustique. 
Dans cette analogie : les insectes = les PODs, les personnes = les nodes 

On associe : 
* une _Toleration_ à un pod
* un _Taint_ à un node

Quand on taint un node, on doit specifier l'effet : 
* NoSchedule (le plus utilisé)
* PreferNoSchedule
* NoExecute (celui-ci veut dire que s'il existait des pods qui n'auraient pas dû être schedulés sur ce node alors kube doit les tuer)

Le node n'est pas une resource kube, donc on n'a pas de def file, donc quand on taint on utilise une command imperative `kubectl taint nodes <nodeName> ...`

Remarque : par defaut kube taint le node `Master` pour qu'aucun pod ne soit schedulé dessus.

## NodeSelector & NodeAffinity

> C'est un moyen de spécifier sur quel node on veut déployer un pod via des labels de node

Le node n'est pas une resource kube, donc on n'a pas de def file, donc on utilise une command imperative `kubectl label nodes <nodeName> ...`. 

Ensuite dans le def file du pod on utilise `nodeSelector` (la feature basique) ou `affinity` (la feature usine à gaz)

<details><summary>NodeAffinity est un NodeSelector on crack</summary>

Il existe notamment plusieurs operators (e.g. NotIn en passant une liste de values, Exists qui teste l'existance de la clé uniquement, etc)
À noter qu'il existe des types de comportement NodeAffinity qui ressemblent à ceux de la feature _Taints&Tolerations_ que l'onspecifie via une longue clé requiredDuringSchedulingIgnoredDuringExecution etc
</details>


<details><summary>Node Affinity Types</summary>

La feature NodeAffinity permet en plus de spécifier ce qu'il faut faire au scheduling du pod ET à l'execution. la partie scheduling s'adresse au scehduler cad lorsque le pod est créé. la partie Execution s'applique meme si un pod tourne depuis longtemps, e.g. si on retire le label qui justifiait l'existence du pod sur le node alors kube supprimera le pod.
</details>

# Resources req and limits

* `request` c'est le min alloué. **Il peut etre configuré au niveau du namespace** via la declaration d'un objet `kind: LimitRange`. Par def 0.5 et 256Mi
* `limit` c'est le max. Par defaut 1 cpu et 512Mi. 

Ces 2 infos sont utilisées par le scheduler pour choisir le node. S'il n'existe aucun node avec assez de resource, alors le POD reste en pending.
Si le CPU dépasse la limit spécifié alors kube va **throttle** le container. Si la memoire depasse, le POD sera finalement _terminated_.
Le min cpu configurable = 0.1 (soit = 100m)
/!\ Ca se regle pour **chaque** container dans un pod.

<details>

Dans le monde de Docker, un conteneur peut utiliser autant de resource qu'il le veut. C'est pour parer ce manque que kube prévoit de spécifier une `limit`

Conversion :
* 1 cpu = 1vCPU (AWS) = 1 core (GCP/Azure) = 1 Hyperthread
* 1 cpu = 1000m
</details>
