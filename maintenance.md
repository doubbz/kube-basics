# Maintenance

## Cluster upgrade

* k8s ne supporte que 3 mineures versions en parallele.
* il est reco d'upgrade mineure par mineure
* On tolère certaines diff de version entre les composants (cf. screen)

![](./images/components_versions.png)

Si on veut upgrade, en gros on procéder en 2 étapes :
* on commence par upgrade le master node car le reste peut tourner plutot bien le temps où le master node est down. Lorsque le master node est à nouveau dispo, tout devrait refonctionner normalement.
* on upgrade ensuite les worker nodes un par un ou bien ajouter des worker nodes dans la bonne version directement avant de supprimer les anciens nodes

Note : `cat /etc/*release*` pour connaitre l'OS et sa version sur le master node et ainsi appliquer la bonne [documentation k8s pour upgrade un cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)

<details>
Plus précisément, si on a kubeadm on peut suivre ces étapes. 
Sur le master node :
* `kubectl upgrade plan` pour voir les infos d'upgrade possible. 
* install la bonne version de `kubeadm` via `apt` 
  * `apt-get upgrade -y kubeadm=1.xx.x-xx`
* lancer `kubeadm upgrade apply v1.xx.x`
* il faut ensuite updgrade les kubelets de chaque node (dont le master)
  * `apt-get upgrade -y kubelet=1.xx.x-xx`
  * `systemctl restart kubelet`

Sur les worker nodes la manip est similaire à qqs details près :
* on drain le node via la master `kubectl drain node-a`
* puis sur le node, on run
  * `apt-get upgrade -y kubeadm=...`
  * `apt-get upgrade -y kubelet=...`
  * `kubeadm upgrade node config --kubelet-version v1.xx.x`
  * `systemctl restart kubelet`
* enfin on "décordonne" le node via la master `kubectl uncordon node-a`

</details>

## Backup et restore de cluster

Premièrement la bonne pratique c'est de tout faire en declaratif et non en imperatif (approche gitops)

Sinon, pour sauvegarder la conf kube comme backup en passant par kube-apiserver :
`kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml`
Mais il existe des outils qui permettent de l'automatiser directement comme velerio (ex ark by heptio)

Enfin, un bon moyen de backup son cluster c'est d'utiliser les features de backup/restore de ETCD vu que toute la conf du cluster kube est dedans :

1. `ETCDCTL_API=3 etcdctl snapshot save snapshot.db` en complétant avec les options --endpoints --cacert --cert et --key que l'on retrouve dans la description du pod etcd
2. `service kupe-apiserver stop` (à vérifier)
3. `ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup` (à noter que l'on doit créer une nouvelle _location_ à chaque fois car sous le capot etcd initialise un nouveau cluster sinon des éléments pourraient se mélanger)
4. il faut ensuite edit le pod etcd pour utiliser la nouvelle location sans oublier de modifier le directory de montage du volume etcd-data. Pour ca on peut editer le yaml `/etc./kubernetes/manifests/etcd.yaml`, vu qu'il s'agit d'un static pod kube-system (il faudra relancer les binaires si k8s n'a pas été installé via kubeadm)
5. `systemctl daemon-reload && service etcd restart && service kube-apiservice start`
