# Setting up Kubernetes clusters

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Setting up Kubernetes clusters](#setting-up-kubernetes-clusters)
	- [Local cluster using Kubespray and Vagrant](#local-cluster-using-kubespray-and-vagrant)
		- [Change base OS](#change-base-os)
		- [Fixing connectivity issues with `kube proxy` and `Vagrant`](#fixing-connectivity-issues-with-kube-proxy-and-vagrant)
		- [Starting the cluster](#starting-the-cluster)
		- [Administration](#administration)
		- [Deploy and run a sample voting app](#deploy-and-run-a-sample-voting-app)
		- [Persistent storage](#persistent-storage)
			- [Create a storage class for local storage](#create-a-storage-class-for-local-storage)
			- [Register local persistent volume](#register-local-persistent-volume)
			- [Claim storage on persistent volumes](#claim-storage-on-persistent-volumes)
		- [Installing and configuring the `Helm` package manager](#installing-and-configuring-the-helm-package-manager)
		- [Using `Terraform` to manage resources on your cluster](#using-terraform-to-manage-resources-on-your-cluster)
		- [Using `kompose` to translate `Docker Compose` files to `Kubernetes`](#using-kompose-to-translate-docker-compose-files-to-kubernetes)

<!-- /TOC -->

## Local cluster using Kubespray and Vagrant

Refer to https://github.com/kubernetes-incubator/kubespray/ for more information.

```
$ git clone https://github.com/kubernetes-incubator/kubespray.git
$ cd kubespray
```

### Change base OS

Change your base OS to CentOS.

```
$ echo "\$os = \"centos\"" >> vagrant/config.rb
```

### Fixing connectivity issues with `kube proxy` and `Vagrant`

You will have issues with connecting to cluster services using kube proxy across multiple hosts.

You need to update the `flannel` configuration such that networking combined with `Vagrant` works properly.

Modify `kubespray/inventory/sample/group_vars/k8s-cluster.yml` and add the following variable.

```yaml
flannel_interface: eth1
```

### Starting the cluster

```
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

Please refer to [sample-app.md](sample-app.md) for installing a sample voting app on your cluster.


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

### Installing and configuring the `Helm` package manager

Helm helps you manage Kubernetes applications — Helm Charts helps you define, install, and upgrade even the most complex Kubernetes application.

Charts are easy to create, version, share, and publish — so start using Helm and stop the copy-and-paste madness.

The latest version of Helm is maintained by the `CNCF` - in collaboration with `Microsoft`, `Google`, `Bitnami` and the `Helm contributor community`.

Also refer to https://docs.helm.sh/ for documentation.

In this case we'll installing Helm on one of the nodes. However, you're also able to install it on your host machine.

Please refer to [helm.md](helm.md) for installing and configuring `Helm`

### Using `Terraform` to manage resources on your cluster

While you could use `kubectl` or similar CLI-based tools mapped to API calls to manage all Kubernetes resources described in YAML files, orchestration with Terraform presents a few benefits.

Please refer to https://www.terraform.io/docs/providers/kubernetes/guides/getting-started.html for more information.

Please refer to [terraform.md](terraform.md) for managing resources using `Terraform`.

### Using `kompose` to translate `Docker Compose` files to `Kubernetes`

`kompose` is a tool to help users who are familiar with `docker-compose` move to [Kubernetes](http://kubernetes.io). `kompose` takes a Docker Compose file and translates it into Kubernetes resources.

`kompose` is a convenience tool to go from local Docker development to managing your application with Kubernetes. Transformation of the Docker Compose format to Kubernetes resources manifest may not be exact, but it helps tremendously when first deploying an application on Kubernetes.

Please refer to [kompose.md](kompose.md) for translating `Docker Compose` to `Kubernetes`.
