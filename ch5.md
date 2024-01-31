# Services: enabling clients to discover and talk to pods

To expose a service we cannot use the IP adress of a pod because:
- pods are ephemeral,
- IP is assigned after scheduling the pod to a node,
- we may have a ReplicaSet of pods.

## Introducing services

A K8s Service is a resource you create to make a single, constant point of entry to a group of pods providing the same service.

Each service has an IP and port that never change.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

Now the Service exposes all pods matched at its own `CLUSTER-IP`.

To test it let's execute `curl` inside an existing pod
```sh
$ kubectl exec kubia-7nog1 -- curl -s http://10.111.249.153
```

Every time a request is sent to a random pod of the Service.
If we want the same client to go to the same pod, we must add to spec:
```yaml
sessionAffinity: ClientIP
```
The default is `None`.

If we want to expose multiple ports we must specify a name for each one
```yaml
ports:
  - name: http
    port: 80
    targetPort: 8080 # container port
  - name: https
    port: 443
    targetPort: 8443 # container port
```

If we name ports on Pods we can use them as `targetPort`:

```yaml
kind: Pod
spec:
  containers:
  - name: kubia
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```
then
```yaml
ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
```


When a pod is started, K8s initializes a set of env variables pointing to each service that exists at that moment.
If we have a `kubia` Service and we create a pod, that the env of the pod will have
```sh
KUBIA_SERVICE_HOST=10.111.249.153
KUBIA_SERVICE_PORT=80
```

We can also use K8s internal DNS.
In the `kube-system` namespace we have `kube-dns` Pod and Service.

We can connect to a Service using its FullyQualifiedDomainName `service-name.namespace.svc.cluster.local`.
`svc.cluster.local` is a configurable cluster domain suffix used in all cluster local service names.
We can omit the suffix, and event the `namespace` if the Pod and the Service are in the same one.

The client must still know the service’s port number.
The Service IP it's a virtual IP and without the port it's nothing. That's why it does not reply to `ping`.

## Connecting to services living outside the cluster

