# Pods: running containers in Kubernetes

Pods are the central concept in k8s: everything else either manages, exposes, or is used by pods.

## Introducing pods

A pod contains one or mode containers.
All of its containers are run on a single worker node.

Every container should run just one single process.
Related processes should be grouped together in one pod.

K8s configures Docker to have all containers of a pod share the same set of Linux namespaces (Network and UTS namespaces).

All containers in a pod share the same port space, and they can communicate through localhost.
Each pod has it's own IP which is unique on the flat cluster network.

A pod is the basic unit of scaling.

## Creating pods from YAML or JSON descriptors

```sh
$ kubectl get pods <POD_NAME> -o yaml
$ kubectl get pods <POD_NAME> -o json
```
give you the yaml or json that defines the pod.

Structure of a pod YAML:
- K8s API version
- Type of K8s object/resource
- Pod metadata: name, namespace, labels, other info
- Pod specification/contents: description of the pod's contents
- Pod status: current info about running pod

No need to provide the status when creating a pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

To get information about things
```sh
$ kubectl explain pods
$ kubectl explian pods.spec
```

To create a pod from YAML
```sh
$ kubectl create -f kubia-manual.yaml
```

```sh
$ kubectl get pods
$ kubectl logs kubia-manual

$ kubectl logs kubia-manual -c kubia # specify container name for a multicontainer pod
```

```sh
$ kubectl port-forward kubia-manual 8888:8080
$ curl localhost:8888
You’ve hit kubia-manual
```


## Organizing pods with labels

A label is an arbitrary key-value pair you attach to a resource, which is then utilized when selecting resources using label selectors.

A resource can have more than one lable, as long as the keys of those labels are unique within that resource.

Add this to `metadata`
```yaml
labels:
    creation_method: manual
    env: prod
```

```sh
$ kubectl get pods --show-labels

$ kubectl get pods -L creation_method,env  # filter
```

We can also add labels to existing pods
```sh
$ kubectl label pods kubia-manual creation_method=manual
```

or override existing ones
```sh
$ kubectl label pods kubia-manual env=debug --overwrite
```


## Listing subsets of pods through label selectors

With label selectors you can select a subset of the resources and perform an operation on it.

The resources can be selected base on whether the resource:
- contains (or diesn't) a label with a certain key;
- contains a label with a certain key and value;
- contains a label with a certain key but not a certain value.

```sh
$ kubectl get pods -l creation_method=manual
$ kubectl get pods -l 'creation_method!=manual'
$ kubectl get pods -l env
$ kubectl get pods -l '!env'
$ kubectl get pods -l 'env in (prod,devel)'
$ kubectl get pods -l 'env notin (prod,devel)'
```

A selector can also include multiple comma-separated criteria
```sh
$ kubectl get pods -l app=pc,rel=beta
```

## Using labels and selectors to constrain pod scheduling

Labels can be attached to any k8s object, including nodes.
The ops team attaches labels specifying the type of hardware any node provides.

```sh
$ kubectl label node gke-kubia-85f6-node-0rrx gpu=true
```

Then you add this to the `spec` of the pod's yaml
```yaml
nodeSelector:
    gpu: "true"
```

We can also assign a pod to a specific node by using the node's unique tag `kubernetes.io/hostname`.


## Annotating pods

Annotations are like labels, but can't be used as selectors and usually store longer pieces of information.

Most of them are automatically added by k8s, but can also be added manually
```sh
$ kubectl annotate pod kubia-manual mycompany.com/someannotation="foo bar"
```

## Using namespaces to group resources

Resource names only need to be unique within a namespace.

Few resources aren't namespaces, like nodes.

```sh
$ kubectl get namespaces
$ kubectl get ns

NAME          LABELS    STATUS    AGE
default       <none>    Active    1h
kube-public   <none>    Active    1h
kube-system   <none>    Active    1h

$ kubectl get po --namespace kube-system
$ kubectl get po -n kube-system
```

A namespace is a k8s resource
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```
and then `create -f`, or
```sh
$ kubectl create namespace custom-namespace
```

To assigne a resource to a namespace, either

- add `namespace: custom-namespace` to `metadata`;
- `kubectl create` adding `-n custom-namespace`.

The following is an alias
```sh
alias kcd='kubectl config set-context $(kubectl config current- context) --namespace '
```
to change namespaces with
```sh
$ kcd some-namespace
```

If a pod in namespace foo knows the IP address of a pod in namespace bar, there is nothing preventing it from sending traffic, such as HTTP requests, to the other pod.


## Stopping and removing pods

```sh
$ kubectl delete po kubia-gpu # with pod names
pod "kubia-gpu" deleted

$ kubectl delete po -l creation_method=manual # with label selection
pod "kubia-manual" deleted
pod "kubia-manual-v2" deleted
```
