# Setting up Kubernetes clusters

## Local cluster using Kubespray and Vagrant

```
$ git clone https://github.com/kubernetes-incubator/kubespray.git
$ cd kubespray
$ vagrant up
```

### Administration

Also refer to https://kubernetes.io/docs/reference/kubectl/cheatsheet/ for more information.

Getting cluster information.
```
vagrant@k8s-01:~$ kubectl cluster-info
Kubernetes master is running at http://localhost:8080
KubeDNS is running at http://localhost:8080/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at http://localhost:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Forwarding port 8001 on the host to 8001 on the guest
```
$ echo '$forwarded_ports = {8001 => 8001}' >> vagrant/config.rb
$ vagrant reload
$ vagrant provision
```

Setting up a proxy to the cluster on port 8001
```
$ vagrant ssh k8s-01
vagrant@k8s-01:~$ kubectl proxy --port=8001 --api-prefix=/ --address='0.0.0.0'
```

Give the dashboard admin access for development purposes.

Also refer to https://github.com/kubernetes/dashboard/wiki/Access-control for more information.

You can grant full admin privileges to Dashboard's Service Account by creating below `ClusterRoleBinding`. Copy the YAML file based on chosen installation method and save as, i.e. `dashboard-admin.yaml`. Use `kubectl create -f dashboard-admin.yaml` to deploy it. Afterwards you can use Skip option on login page to access Dashboard.

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

Browse to the dashboard at
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login and click on "Skip"

### Deploy and run a sample voting app

Get the sources and deploy the sample app
```
vagrant@k8s-01:~$ git clone https://github.com/dockersamples/example-voting-app.git
vagrant@k8s-01:~$ cd example-voting-app
vagrant@k8s-01:~$ kubectl create -f k8s-specifications/
```

The vote interface is then available on port 31000 on each host of the cluster, the result one is available on port 31001.

Architecture
-----

![Architecture diagram](architecture.png)

* A Python webapp which lets you vote between two options
* A Redis queue which collects new votes
* A .NET worker which consumes votes and stores them in…
* A Postgres database backed by a Docker volume
* A Node.js webapp which shows the results of the voting in real time

Display deployment information on the cluster
```
vagrant@k8s-01:~/example-voting-app$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
db           ClusterIP   10.233.35.241   <none>        5432/TCP         11m
kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP          7d
redis        ClusterIP   10.233.57.65    <none>        6379/TCP         11m
result       NodePort    10.233.37.243   <none>        5001:31001/TCP   11m
vote         NodePort    10.233.62.66    <none>        5000:31000/TCP   11
```
