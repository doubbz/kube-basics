# Cheatsheets

## InitContainers

* si on veut run un truc avant le lancement du/des container(s) d'un pod, on peut utiliser initContainers qui est une liste de command. La liste s'execute séquentiellement et kube attend qu'elles soient toutes terminées avec succès pour lancer le(s) container

## YAML definitions files

4 clés sont à la racine : 
* `apiVersion`, `kind` >> ces 2 là sont assez vite vus car il y a pas 300 solutions (cf. screen) 

![pod_yaml_kind_version](images/pod_yaml_kind_version.png)

* `metadata` >> où l'on peut annoter des choses concernant le pod, notamment la clé `labels` qui permet en général de distinguer les pods sur le cluster (le choix des clés est libre pour tout ce qui se trouve dans les labels)
* `spec` >> c'est là que tout va se jouer. À noter que les clés possibles dans cette partie sont différentes selon ce qu'on a choisi de déployer i.e. ce qu'on a mis dans `apiVersion`, `kind`
  * `containers` est une liste car on peut avoir plusieurs containers dans un pod. C'est ici qu'on va renseigner l'image docker sur le registry docker hub (on peut utiliser un autre regitry bien sûr, dans ce cas il faut mettre l'url en entier)

## CLI commands

* pour créer un pod dont le conteneur expose le port 8080 et est exposé via un service clusterIP
```sh
kubectl run pod-name -i <image-name> --port=8080 --expose
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
kubectl create deployment <name> --image=<image> --replicas=<number> --dry-run=client -o yaml > deployment-auto-gen.yaml
```

* si on est ssh sur le master node, on peut toujours se ssh sur un worker node via l'ip interne de kube i.e. celle qui commence par `10.*`

* pour modifier les valeurs d'un context pré-existant (p ex le namespace)
`kubectl config set-context $(kubectl config current-context) --namespace=default`

* rollout commands pour avoir des infos sur les revisions ou bien pour **rollback**

```sh
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deployment/myapp-deployment # pour lister les revisions d'une ressource
kubectl rollout undo ... # pour rollback
```

* pour "vider" un node worker (avant un opé de maintenance p ex) cad bouger les pods existants dans les autres nodes disponibles
```sh
# vide le node
kubectl drain <node-name>
# re-remplir le node
kubectl uncordon <node-name>
# pour bloquer le scheduling de nouvelles ressources sur un node
kubectl cordon <node-name>
```

* pour afficher le contenu d'un certif openssl, on utilise la commande 
```sh 
openssl x509 -in <path du certif> -text -noout
```

* charger la config `kubeconfig` dans curl directement (si on veut parler à l'api http de kube directement)
```sh
k proxy # pour lancer un proxy local qui utilisera la conf kubeconfig sans avoir à passer --cert --key -- cacert tout le temps dans curl\
```

* pour créer un serviceaccount
```sh
k create serviceaccount <name>
```
