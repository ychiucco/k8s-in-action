# Replication and other controllers: deploying managed pods

## Keeping pods healthy

K8s can check if a container is still alive through Liveness Probes.

Three possible way:
- HTTP GET request with a 2xx or 3xx response;
- succeed in opening a TCP Socket connection;
- exec some command inside the container, getting 0 as status code.

This is what it looks like inside the yaml

```yaml
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```

You can also set delay, timeout, period and required failures.

To get the log of a crashed pod, then restarted
```sh
$ kubectl logs mypod --previous
```

Liveness probes shouldn't use too many computational resources and shouldn't take too long to complete.

This job is performed by the Kubelet on the node hosting the pod.
If the node crashes, there it's the ComprolPlan that must create the replacement.

## Introducing ReplicationControllers

A `ReplicationController` is a K8s resource that ensures its pods are always kept running.

Composed of three parts:
- label selector
- replica count
- pod template

You can modify them at any time, but just changes in replica count affect running pods.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
template:
  metadata:
    labels:
      app: kubia
  spec:
    containers:
    - name: kubia
      image: luksa/kubia
      ports:
      - containerPort: 8080
```

Specifying the label selector of the ReplicationController is not recommended, as it is inferred from the label of the pod.


```sh
$ kubectl get replicationcontroller
$ kubectl get rc
```

A pod is not tied to its ReplicationController, but has a reference to it at `metadata.ownerReferences`.

To edit the yaml of a ReplicationController on vim
```sh
$ kubectl edit rc kubia
```

Shortcut to scale up/down without edit
```sh
$ kubectl scale rc kubia --replicas=10
```

When you delete a ReplicationController through `kubectl delete`, the pods are also deleted. To keep them alive
```sh
$ kubectl delete rc kubia --cascade=false
```


# Using ReplicaSets instead of ReplicationControllers

ReplicationControllers have been replaced by ReplicaSets and will eventually be deprecated.

Usually ReplicaSets are created within the creation of Deployments.

ReplicaSets are just like Controllers but they have more expressive pod selectors:
- match pods without a certain label
- match pods with a certain key, regardless of its value

```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```

```sh
$ kubectl get replicaset
$ kubectl get rs
```

Other possible selectors
```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - kubia
```
Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`.

We can specify multiple expressions and all of them must be true.
We can use both `matchLabels`and `matchExpressions


## Running exactly one pod on each node with DaemonSets

Sometimes you need to run exactly one instance of the same pod on each node.
That is what DaemonSets are used for. They're meant to run system services.

You can also specify a `nodeSelector` in the pod template, to avoid certain nodes.

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```

```sh
$ kubectl get daemonsets
$ kubectl get ds
```

## Running pods that perform a single compleatable task

Jobs are resources used when we want to perform an ad-hoc task, and we don't want the container to restart if the Pod status is `Completed`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

`restartPolicy` default is `Always`, but for Jobs we need to set `OnFailure` or `Never`.

```sh
$ kubectl get jobs
```

If you want the Job to run successfully many times, you add this to the spec
```yaml
completions: 5
```
This will run 5 pods, one after the completion of the previous one.

To have them running in parallel add this to the spec
```yaml
parallelism: 2
```
To change parallelism when the Job is running
```sh
$ kubectl scale job multi-completion-batch-job --replicas 3
```

Add `spec.activeDeadlineSeconds` to set the maximum time a pod has to complete its process.
Add `spec.backoffLimit` to configure how many times a Job can be retried before being marked as failed (default: 6).


## Scheduling Jobs to run periodically or once in the future

We can also create CronJobs
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```

By adding `spec.startingDeadlineSeconds` we set the maximum amount of seconds we can wait the CronJob to run after the scheduled time.
