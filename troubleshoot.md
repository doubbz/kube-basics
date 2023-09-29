Pour checker les logs, il peut y avoir 3 façons :
```sh
kubectl logs etcd-master
# ou bien
journalctl -u etcd.service -l # qui permet de visualiser les logs systemd => nécessaire notamment quand kube a été instal à la mano
```
Sauf que si un des core components est down, alors il se peut que kubectl ne réponde pas. Dans ce cas, il faut essayer via docker
```sh
docker ps -a | grep etcd-master # pour retrouver le conteneur qui nous intéresse
docker logs $(docker ps -a | grep etcd-master | cut -d " " -f1)
```

si docker n'est pas installé, tester via l'interface container runtime
```sh
crictl ps -a
crictl logs <container id>
```
