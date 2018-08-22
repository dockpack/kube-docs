### Installing `kompose`

```
$ vagrant ssh k8s-01
[vagrant@k8s-01 ~]$ sudo yum -y install epel-release
[vagrant@k8s-01 ~]$ sudo yum -y install kompose
```

### Converting to `Kubernetes` and adding it to your cluster
```
$ vagrant ssh k8s-01
[vagrant@k8s-01 ~]$ kompose convert --file docker-voting.yaml -o k8s-docker-voting.yaml
[vagrant@k8s-01 ~]$ kubectl create -f k8s-docker-voting.yaml
```
