### Installing `Kubeless` on your cluster

Please refer to https://kubeless.io/docs/quick-start/ for more information.

This below is a show case of deploying kubeless to a non-RBAC Kubernetes cluster.

```
$ vagrant ssh k8s-01
[vagrant@k8s-01 ~]$ export RELEASE=$(curl -s https://api.github.com/repos/kubeless/kubeless/releases/latest | grep tag_name | cut -d '"' -f 4)
[vagrant@k8s-01 ~]$ kubectl create ns kubeless
[vagrant@k8s-01 ~]$ kubectl create -f https://github.com/kubeless/kubeless/releases/download/$RELEASE/kubeless-$RELEASE.yaml

[vagrant@k8s-01 ~]$ kubectl get pods -n kubeless
NAME                                           READY     STATUS    RESTARTS   AGE
kubeless-controller-manager-567dcb6c48-ssx8x   1/1       Running   0          1h

[vagrant@k8s-01 ~]$ kubectl get deployment -n kubeless
NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubeless-controller-manager   1         1         1            1           1h

[vagrant@k8s-01 ~]$ kubectl get customresourcedefinition
NAME                          AGE
cronjobtriggers.kubeless.io   1h
functions.kubeless.io         1h
httptriggers.kubeless.io      1h
```

For installing `kubeless` CLI using execute:

```
[vagrant@k8s-01 ~]$ export OS=$(uname -s| tr '[:upper:]' '[:lower:]')
[vagrant@k8s-01 ~]$ curl -OL https://github.com/kubeless/kubeless/releases/download/$RELEASE/kubeless_$OS-amd64.zip && \
  unzip kubeless_$OS-amd64.zip && \
  sudo mv bundles/kubeless_$OS-amd64/kubeless /usr/bin/
```

### Deploying a sample serverless function

Copy the sample function [test.py](kubeless/test.py) to your cluster.

```python
def hello(event, context):
  print event
  return event['data']
```

Functions in Kubeless have the same format regardless of the language of the function or the event source. In general, every function:

- Receives an object `event` as their first parameter. This parameter includes all the information regarding the event source. In particular, the key 'data' should contain the body of the function request.
- Receives a second object `context` with general information about the function.
- Returns a string/object that will be used as response for the caller.

You can find more details about the function interface [here](https://kubeless.io/docs/kubeless-functions#functions-interface)

You create it with:

```
[vagrant@k8s-01 ~]$ kubeless function deploy hello --runtime python2.7 \
                                --from-file test.py \
                                --handler test.hello
INFO[0000] Deploying function...
INFO[0000] Function hello submitted for deployment
INFO[0000] Check the deployment status executing 'kubeless function ls hello'
```

Let's dissect the command:

- `hello`: This is the name of the function we want to deploy.
- `--runtime python2.7`: This is the runtime we want to use to run our function. Available runtimes are shown in the help information.
- `--from-file test.py`: This is the file containing the function code. It is supported to specify a zip file as far as it doesn't exceed the maximum size for an etcd entry (1 MB).
- `--handler test.foobar`: This specifies the file and the exposed function that will be used when receiving requests. In this example we are using the function `foobar` from the file `test.py`.
You can find the rest of options available when deploying a function executing `kubeless function deploy --help`

You will see the function custom resource created:

```
[vagrant@k8s-01 ~]$ kubectl get functions
NAME         AGE
hello        1h

[vagrant@k8s-01 ~]$ kubeless function ls
NAME            NAMESPACE   HANDLER       RUNTIME   DEPENDENCIES    STATUS
hello           default     helloget.foo  python2.7                 1/1 READY

```

You can then call the function with:

```
[vagrant@k8s-01 ~]$ kubeless function call hello --data 'Hello world!'
Hello world!
```

### Cleanup

You can delete the function as follows:

```
[vagrant@k8s-01 ~]$ kubeless function delete hello
```
