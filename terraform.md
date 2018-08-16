### Installing `Terraform`

```
$ vagrant ssh k8s-01
vagrant@k8s-01:~$ wget https://releases.hashicorp.com/terraform/0.11.8/terraform_0.11.8_linux_amd64.zip
vagrant@k8s-01:~$ unzip terraform_0.11.8_linux_amd64.zip
vagrant@k8s-01:~$ sudo mv ./terraform /usr/bin/terraform
```

### Setup your `Kubernetes` provider
Use [_providers.tf](terraform/_providers.tf) for setting up your `Kubernetes` provider.

```
vagrant@k8s-01:~$ terraform init
...
* provider.kubernetes: version = "~> 1.2"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

### Deploy a sample `nginx` application

Ensure [nginx.tf](terraform/nginx.tf) exists.

```
vagrant@k8s-01:~$ terraform apply
...
kubernetes_service.nginx: Creation complete after 0s (ID: default/nginx-example)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

You can access the application at http://localhost:8001/api/v1/namespaces/default/services/nginx-example:80/proxy/

Refer to https://www.terraform.io/docs/providers/kubernetes/index.html for more information.
