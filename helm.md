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
