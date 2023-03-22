# Cheatsheets

## YAML definitions files

4 clés sont à la racine : 
* `apiVersion`, `kind` >> ces 2 là sont assez vite vus car il y a pas 300 solutions (cf. screen) 

![pod_yaml_kind_version](images/pod_yaml_kind_version.png)

* `metadata` >> où l'on peut annoter des choses concernant le pod, notamment la clé `labels` qui permet en général de distinguer les pods sur le cluster (le choix des clés est libre pour tout ce qui se trouve dans les labels)
* `spec` >> c'est là que tout va se jouer. À noter que les clés possibles dans cette partie sont différentes selon ce qu'on a choisi de déployer i.e. ce qu'on a mis dans `apiVersion`, `kind`
  * `containers` est une liste car on peut avoir plusieurs containers dans un pod. C'est ici qu'on va renseigner l'image docker sur le registry docker hub (on peut utiliser un autre regitry bien sûr, dans ce cas il faut mettre l'url en entier)

## CLI commands

* pour créer un pod sans passer par un controller
```sh
kubectl run pod-name -i <image-name>
```

* Pour appliquer un _def file_
```sh
kubectl create -f <path-to-yaml>
```
ou
```sh
kubectl apply -f <path-to-yaml>
```
qui fonctionne tout le temps i.e. pour la creation ou l'édition

* pour générer un def file facilement
```sh
kubectl run redis --image=redis123 --dry-run=client -o yaml > redis-pod.yaml
```

* pour generer un conf file de type Deployment p ex 
```sh
kubectl run --generator=deployment/v1beta1 nginx --image=nginx --dry-run --replicas=4 -o yaml > nginx-deployment.yaml
```

* si on est ssh sur le master node, on peut toujours se ssh sur un worker node via l'ip interne de kube i.e. celle qui commence par `10.*`

* connaitre les replicaset via la command `kubectl get replicas<tab>`
