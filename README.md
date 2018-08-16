# Setting up Kubernetes clusters

## Local cluster using Kubespray and Vagrant

Refer to https://github.com/kubernetes-incubator/kubespray/ for more information.

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

You can grant full admin privileges to Dashboard's Service Account by creating below `ClusterRoleBinding`. Copy the YAML file based on chosen installation method and save as, i.e. [dashboard-admin.yaml](rbac/dashboard-admin.yaml). Use `kubectl create -f dashboard-admin.yaml` to deploy it. Afterwards you can use Skip option on login page to access Dashboard.

Browse to the dashboard at
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login and click on "Skip"

`Note: don't use this configuration for production clusters because of the security risks!`

### Deploy and run a sample voting app

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

### Persistent storage

#### Create a storage class for local storage

A `StorageClass` provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called “profiles” in other storage systems.

Also refer to https://kubernetes.io/docs/concepts/storage/storage-classes/

You can define a storage class for local storage by creating below `StorageClass`. Copy the YAML file based on chosen installation method and save as, i.e. [local-sc.yaml](storage/local-sc.yaml). Use `kubectl create -f local-sc.yaml` to deploy it.

#### Register local persistent volume

You can allocate some space for persistent storage by creating below `PersistentVolume`. Copy the YAML file based on chosen installation method and save as, i.e. [local-pv.yaml](storage/local-pv.yaml). Use `kubectl create -f local-pv.yaml` to deploy it.

`Note: don't use this configuration for production clusters!`

Also refer to https://kubernetes.io/docs/concepts/storage/persistent-volumes/

#### Claim storage on persistent volumes

You can claim some space for persistent storage by creating below `PersistentVolumeClaim`. Copy the YAML file based on chosen installation method and save as, i.e. [local-pvc.yaml](storage/local-pvc.yaml). Use `kubectl create -f local-pvc.yaml` to deploy it.

### Fixing connectivity issues with `kube proxy`

You might have issues with connecting to cluster services using kube proxy.
Executing the following commands as a workaround on each node will fix these issues.

```
# iptables -R KUBE-FIREWALL 1 -j ACCEPT
# iptables -R KUBE-SERVICES 1 -j ACCEPT
# iptables -R KUBE-SERVICES 2 -j ACCEPT
# apt-get -y install netfilter-persistent
# service netfilter-persistent start
```

`TODO: find a better way of fixing these issues :)`

### Installing and configuring the `Helm` package manager

Helm helps you manage Kubernetes applications — Helm Charts helps you define, install, and upgrade even the most complex Kubernetes application.

Charts are easy to create, version, share, and publish — so start using Helm and stop the copy-and-paste madness.

The latest version of Helm is maintained by the `CNCF` - in collaboration with `Microsoft`, `Google`, `Bitnami` and the `Helm contributor community`.

Also refer to https://docs.helm.sh/ for documentation.

In this case we'll installing Helm on one of the nodes. However, you're also able to install it on your host machine.

```
$ vagrant ssh k8s-01
vagrant@k8s-01:~$ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
vagrant@k8s-01:~$ tar xzf helm-v2.9.1-linux-amd64.tar.gz
vagrant@k8s-01:~$ cd linux-amd64/
vagrant@k8s-01:~$ sudo mv ./helm /usr/bin/helm
```

#### Configuring RBAC for `Helm` and `Tiller`

Tiller is a server-side component for Helm. You can create a namespace for Tiller and configure role-based access control. Tiller will then be restricted to deploying resources only in that namespace.

You can specify a `Role` and `RoleBinding` to limit Tiller’s scope to a particular namespace.

Also refer to https://docs.helm.sh/using_helm/#tiller-and-role-based-access-control for more information.

```
vagrant@k8s-01:~$ kubectl create namespace tiller-world
namespace "tiller-world" created
vagrant@k8s-01:~$ kubectl create serviceaccount tiller --namespace tiller-world
serviceaccount "tiller" created
```

Define a Role that allows `Tiller` to manage all resources in tiller-world like in [role-tiller.yaml](helm/role-tiller.yaml):

```
vagrant@k8s-01:~$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

Use [rolebinding-tiller.yaml](helm/rolebinding-tiller.yaml) to create your binding.

```
vagrant@k8s-01:~$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```

Afterwards you can run helm init to install Tiller in the `tiller-world` namespace.

```
vagrant@k8s-01:~$ helm init --service-account tiller --tiller-namespace tiller-world
Creating /home/vagrant/.helm
Creating /home/vagrant/.helm/repository
Creating /home/vagrant/.helm/repository/cache
Creating /home/vagrant/.helm/repository/local
Creating /home/vagrant/.helm/plugins
Creating /home/vagrant/.helm/starters
Creating /home/vagrant/.helm/cache/archive
Creating /home/vagrant/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /home/vagrant/.helm.
Warning: Tiller is already installed in the cluster.
(Use --client-only to suppress this message, or --upgrade to upgrade Tiller to the current version.)
Happy Helming!
```

Add the incubator repository for Helm.

```
vagrant@k8s-01:~$ helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
```

#### Install `Jenkins` on your Kubernetes cluster using `Helm`.

Create a persistent volume for Jenkins. Use [jenkins-pv.yaml](jenkins/jenkins-pv.yaml) for creating the `PersistentVolume`.

```
vagrant@k8s-01:~$ kubectl create -f jenkins-pv.yml --namespace tiller-world
```

Create a persistent volume claim for Jenkins. Use [jenkins-pvc.yaml](jenkins/jenkins-pvc.yaml) for creating the `PersistentVolumeClaim`.

```
vagrant@k8s-01:~$ kubectl create -f jenkins-pvc.yml --namespace tiller-world
```

Install Jenkins using Helm.

```
vagrant@k8s-01:~$ helm install --name jenkins -f jenkins-values.yaml --namespace tiller-world --tiller-namespace tiller-world --debug stable/jenkins
```

Fetch your Jenkins admin password.

```
vagrant@k8s-01:~$ printf $(kubectl get secret --namespace tiller-world jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

Browse to your Jenkins instance on the cluster.

http://localhost:8001/api/v1/namespaces/tiller-world/services/jenkins:http/proxy/
