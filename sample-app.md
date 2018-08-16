Get the sources and deploy the sample app
```
vagrant@k8s-01:~$ git clone https://github.com/dockersamples/example-voting-app.git
vagrant@k8s-01:~$ cd example-voting-app
vagrant@k8s-01:~$ kubectl create -f k8s-specifications/
```

The vote interface is then available on port `31000` on each host of the cluster, the result one is available on port `31001`.

Architecture
-----

![Architecture diagram](architecture.png)

* A Python webapp which lets you vote between two options
* A Redis queue which collects new votes
* A .NET worker which consumes votes and stores them in…
* A Postgres database backed by a Docker volume
* A Node.js webapp which shows the results of the voting in real time

### Display deployment information on the cluster

```
vagrant@k8s-01:~/example-voting-app$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
db           ClusterIP   10.233.35.241   <none>        5432/TCP         11m
kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP          7d
redis        ClusterIP   10.233.57.65    <none>        6379/TCP         11m
result       NodePort    10.233.37.243   <none>        5001:31001/TCP   11m
vote         NodePort    10.233.62.66    <none>        5000:31000/TCP   11
```

### Forwarding ports `31000` and `31001` from the host to the cluster

Update `vagrant/config.rb` with the following contents.

```ruby
$forwarded_ports = {8001 => 8001, 31000 => 31000, 31001 => 31001}
```

### Restart the cluster

```
$ vagrant reload
$ vagrant provision
```

### Allow access to the dashboard

```
$ vagrant ssh k8s-01
vagrant@k8s-01:~$ kubectl create -f dashboard-admin.yml
clusterrolebinding.rbac.authorization.k8s.io "kubernetes-dashboard" created
```

### Start the proxy
```
vagrant@k8s-01:~$ kubectl proxy --port=8001 --api-prefix=/ --address='0.0.0.0'
```

### Recreate the sample app
```
vagrant@k8s-01:~$ cd example-voting-app/
vagrant@k8s-01:~/example-voting-app$ kubectl delete -f k8s-specifications/
vagrant@k8s-01:~/example-voting-app$ kubectl create -f k8s-specifications/
```
