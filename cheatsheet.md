# Cheatsheets

## InitContainers

* si on veut run un truc avant le lancement du/des container(s) d'un pod, on peut utiliser initContainers qui est une liste de command. La liste s'execute séquentiellement et kube attend qu'elles soient toutes terminées avec succès pour lancer le(s) container

## YAML definitions files

4 clés sont à la racine : 
* `apiVersion`, `kind` >> ces 2 là sont assez vite vus car il y a pas 300 solutions (cf. screen) 

![pod_yaml_kind_version](images/pod_yaml_kind_version.png)

* `metadata` >> où l'on peut annoter des choses concernant le pod, notamment la clé `labels` qui permet en général de distinguer les pods sur le cluster (le choix des clés est libre pour tout ce qui se trouve dans les labels)
* `spec` >> c'est là que tout va se jouer. À noter que les clés possibles dans cette partie sont différentes selon ce qu'on a choisi de déployer i.e. ce qu'on a mis dans `apiVersion`, `kind`
  * `containers` est une liste car on peut avoir plusieurs containers dans un pod. C'est ici qu'on va renseigner l'image docker sur le  docker hub (on peut utiliser un autre regitry bien sûr, dans ce cas il faut mettre l'url en entier)

## CLI commands

* pour créer un pod dont le conteneur expose le port 8080
```sh
kubectl run pod-name -i <image-name> --port=8080
```

* pour créer un pod dont le conteneur expose le port 8080 ET EN PLUS est exposé via un service clusterIP
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
ou
```sh
# wildcards pour filtrer
k apply -f '<prefix>*.yaml'
```

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
Les infos les plus courantes sont : `Subject`, `Subject Alternative Name`, `Issuer` et `Validity`


* charger la config `kubeconfig` dans curl directement (si on veut parler à l'api http de kube directement)
```sh
k proxy # pour lancer un proxy local qui utilisera la conf kubeconfig sans avoir à passer --cert --key -- cacert tout le temps dans curl\
```

* pour créer un serviceaccount
```sh
k create serviceaccount <name>
```

* pour lister les PV d'un cluster
```sh
k get peristentvolumes
```


### Backup and restore

* procédure backup-restore avec etcd
```sh
ETCDCTL_API=3 etcdctl snapshot save snapshot.db --cert= --key= --cacert= --endpoints=
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup --cert= --key= --cacert= --endpoints=
# puis relancer le pod (/etc/kubernetes/manifests/etcd.yaml) ou le service (/etc/systemd/system/etcd.service ou systemctl status etcd pour récupérer le path pour l'editer puis service etcd restart)
```

* pour créer un token jwt pour un SA
```sh
kubectl create token <sa-name>
```

* pour monter automatique le token d'un SA à un pod, utiliser `serviceAccountName: <sa-name>`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  serviceAccountName: build-robot
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```

* pour base64 decode la data d'un secret opaque sans s'emmerder avec des grep et des cut :
```sh
kubectl get secret <name> -o go-template='{{.data.<data_key> | base64decode}}'
```


### debug networking

* pour trouver les infos de l'interface bridge que 
```sh
ip addr show type bridge
```

* pour retrouver l'interface network d'un node
```sh
#1 retrouver l'ip interne attribuée à notre node
k get nodes -o wide
#2 grep cette IP dans la liste des interfaces réseaux
ip a | grep -B3 <l'addr ip>
```

* pour retrouver les ports qu'écoutent les différents pods des core components
```sh
netstat -nlpt
# netstat -npt si on cherche à voir le nombre de connection d'une service
```

* pour retrouver les noms et les raccourcis (ainsi que l'API Group associé) :
```sh
kubectl api-resources
```

* pour compter toutes les ressources (pod, rs, deploy, svc)
```sh
k get all -l <labels or selectors> --no-headers | wc -l
```

* pour créer un daemonset, on peut partir du conf file d'un deployment et remplacer Deployment par DaemonSet (il faudra supprimer replicas & strategy)

* Kube ajoute automatiquement le nom du node à la suite pour nommer un static pod. On peut se servir de ca pour reconnaitre les static pods des autres via la reponse de la commande `kubectl get po`

* k offre quasi-nativemnent du monitoring/logs
```sh
$ k top -h
Display Resource (CPU/Memory) usage.

 The top command allows you to see the resource consumption for nodes or pods.

 This command requires Metrics Server to be correctly configured and working on the server.

Available Commands:
  node          Display resource (CPU/memory) usage of nodes
  pod           Display resource (CPU/memory) usage of pods
```

* pour retrouver le detail d'un process qui tourne sur un host
```sh
systemctl list-unit-files
syetemctl status <unit-files> -l --no-pager
```

* pour retrouver les conf CNI d'un cluster kube
```sh
# Par def c'est dans `/opt/cni/bin/` qu'on retrouve les plugins/binaires disponibles
ls /opt/cni/bin/

# Pour retrouver le plugin cni utilisé
ls /etc/cni/net.d/
```

* use the ipcalc tool to see the network details:
```sh
root@controlplane:~> ipcalc -b 10.33.39.8
Address:   10.33.39.8           
Netmask:   255.255.255.0 = 24   
Wildcard:  0.0.0.255            
=>
Network:   10.33.39.0/24        
HostMin:   10.33.39.1           
HostMax:   10.33.39.254         
Broadcast: 10.33.39.255         
Hosts/Net: 254                   Class A, Private Internet
```

kube-dns est le service qui permet de faire la résolution dns dans le sluster (notamment au niveau de chaque pod c'est donc son ip qui apparaitra dans /etc/resolv.conf)

* pour tester la conf de sécurité sur le component api-servcer (les objets RBAC p ex), on peut utiliser l'option --as (il faut ajouter les creds du user dans le kubeconfig au préalable)
```sh
k <command> --as <username>
```

* un couple de manifests PV PVC sur le host
```yaml

```

* pour debug les dns de pod et svc dans le cluster :
```sh
k run busybox --image=busybox -- sleep 4000
k exec busybox -- nslookup <domain name e.g. svc.namespace.cluster.local>
```

* pour savoir si un svc a bien des terminaisons :
```sh
kubectl get endpoints -n <namespace> <service-name>
```

* port-forward
```sh 
kubectl port-forward service/bns-http-service-kind 8088:http
```

* installer un plugin avec krew
```sh
kubectl krew install score
```

* pour lancer un conteneur temporaire pour se ssh dessus et tester des trucs (notamment coté réseau p ex)
```sh
kubectl run -n <namespace> -i --tty --rm debug --image=busybox -- sh
```

* pour debug un prob réseau entre conteneur, quelques commandes utiles :
```sh
# avec kubectl
k get endpoints -n <NS>

# depuis un conteneur dans le meme NS ou dans un autre
wget -qO- http://<endpoint-ip>:<endpoint-port>
nslookup <my-app-service>.<my-app-namespace>.svc.cluster.local

# depuis un conteneur du pod que l'on souhaite joindre
netstat -nlpt
# il faut que cela écoute sur le port ET vers l'extérieur et donc pas seulement sur 127.0.0.1
```

* on peut utiliser kubectl pour faire des appels API http sécurisé vers api server - particulièrement utile si on utilise une API / un endpoint / un argument / une option qui n'a pas encore été implémenté par kubectl :
```sh
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/<namespace>/pods/*/phpfpm_processes_total"
```

* ajouté un max de verbosité dans kubectl (tellement qu'on voit meme les appels http passés à l'api server):
```sh
kubectl <command> -v 6"
```

* get workload cpu requests in a given namespace
```sh
k get po -n <NAMESPACE> --no-headers=true -o custom-columns='NAME:.metadata.name, CPU_REQUEST:.spec.containers[*].resources.requests.cpu, MEMORY_REQUEST:.spec.containers[*].resources.requests.memory'
```

* la meme donnée mais aggrégée
```sh
k get po -n <NAMESPACE> --no-headers=true -o custom-columns='NAME:.metadata.name, CPU_REQUEST:.spec.containers[*].resources.requests.cpu, MEMORY_REQUEST:.spec.containers[*].resources.requests.memory' | awk 'BEGIN {cpu=0; memory=0} {cpu+=$2; memory+=$3} END {print "Total CPU Requests: " cpu "\nTotal Memory Requests: " memory}'
```

* Exemples de commandes forgées à partir de 2 requêtes
```sh
# commande pour downscale tous les deployments contenant certains labels
kubectl get ns -l part-of=brand-and-shop -o custom-columns=NAME:.metadata.name --no-headers | xargs -n1 kubectl scale deploy -l part-of=brand-and-shop,component=workload,type=worker --replicas 0 -n

# commande pour ajouter des labels
k get rabbitmqclusters.rabbitmq.com -A -o custom-columns=NAME:{.metadata.name} --no-headers | xargs -I {} -n1 kubectl label rabbitmqclusters {} rabbitmq.com/pauseReconciliation=true

# pour reconstituer une liste '<ns>/<name>' puisque c'est ko avec l'operator rabbitmq
k get rabbitmqclusters.rabbitmq.com -A -o custom-columns=NAMESPACE:{.metadata.namespace},NAME:{.metadata.name} --no-headers | awk '{print $1"/"$2}'

# pour labeliser les ressources de meme type i.e. rabbitmqcluster dans des namespaces différents avec le meme label
kubectl get rabbitmqclusters.rabbitmq.com -A -o custom-columns=NAMESPACE:{.metadata.namespace},NAME:{.metadata.name} --no-headers | xargs -I {} bash -c 'set -- {}; kubectl label rabbitmqclusters -n $1 $2 rabbitmq.com/pauseReconciliation=true'
```
  * `xargs -I {}` : Cette commande permet de prendre l'entrée standard (la sortie de la commande précédente) et de l'utiliser comme argument dans une autre commande.
    * Le flag `-I {}` indique qu'à chaque itération, xargs remplacera `{}` par la ligne extraite de l'entrée standard (ici, une combinaison NAMESPACE NAME).
  * `bash -c 'set -- {}; ...'` : Lance un sous-shell avec Bash.
    * `set -- {}` : Cela remplace les arguments positionnels de la commande Bash avec la sortie de la commande précédente. Ici, `{}` est remplacé par la chaîne `"NAMESPACE NAME"`.
    `$1` fait référence au namespace, et `$2` au nom du cluster RabbitMQ.
  * `kubectl label rabbitmqclusters -n $1 $2 rabbitmq.com/pauseReconciliation=true` :
    * Cette commande applique un label spécifique (rabbitmq.com/pauseReconciliation=true) à chaque cluster RabbitMQ trouvé.
    `-n $1` : Spécifie le namespace de la ressource (représenté par `$1`).
    `$2` : Spécifie le nom du cluster RabbitMQ à labelliser.

* penser au kubectl patch pour automatiser des pods exceptionnels tout en s'appuyant sur des pods existants
```sh
kubectl create job bns-job-wrapper -n $NAMESPACE --from=cronjobs/bns-job-reindex-product --dry-run=client -o yaml | kubectl patch --dry-run=client -o yaml --type json --patch "[{ \"op\": \"replace\", \"path\": \"/spec/template/spec/containers/0/command\", \"value\": [\"${COMMAND}\"] }]" -f - | kubectl apply -f -
```

Sinon c'est aussi possible avec helm grace à un `{{- if .Values.jobManual }}` et un `helm install --set jobManual=true`

* pour une liste des docker capabilities / linux capabilities pour les spec.securityContext, voir [ici](https://unofficial-kubernetes.readthedocs.io/en/latest/concepts/policy/container-capabilities/)

* ne pas oublier le cache DNS local sur les ordis quand on teste apres un switch dns : 
```sh
# sur mac
alias flushdns='sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder'
```
et il faut aussi flush le cache du navigateur p ex sur firefox : `about:config`>`network.dnsCacheExpiration`>`value=0`
