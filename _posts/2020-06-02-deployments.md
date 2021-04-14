---
layout: post
title: Deployment Update
comments: false
category: blog
---
# Deployment Update


`Deployment`s are intended to replace `ReplicationController`s.
They provide the same replication functions (through `ReplicaSet`s) and also the ability to rollout changes and roll
them back if necessary.

Let's create a simple `Deployment` using the same image we've been using.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: deployment-rs
  template:
    metadata:
      labels:
        app: deployment-rs
        environment: dev
    spec:
      containers:
      - name: container01
        image: r.deso.tech/library/nginx
        ports:
        - containerPort: 80
```

Now go ahead and create the Deployment:

```bash
kubectl create -f deploy.yaml
```

> ```
> deployment.apps/deployment-test created
> ```

Now let's go ahead and describe the `Deployment`:

```bash
kubectl describe deploy deployment-test
```

> ```
> Name:                   deployment-test
> Namespace:              default
> CreationTimestamp:      Tue, 16 Jul 2019 12:58:13 +0200
> Labels:                 <none>
> Annotations:            deployment.kubernetes.io/revision: 1
> Selector:               app=test-rs
> Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
> StrategyType:           RollingUpdate
> MinReadySeconds:        0
> RollingUpdateStrategy:  25% max unavailable, 25% max surge
> Pod Template:
>   Labels:  app=test-rs
>            environment=dev
>   Containers:
>    container01:
>     Image:        r.deso.tech/library/nginx
>     Port:         80/TCP
>     Host Port:    0/TCP
>     Environment:  <none>
>     Mounts:       <none>
>   Volumes:        <none>
> Conditions:
>   Type           Status  Reason
>   ----           ------  ------
>   Available      True    MinimumReplicasAvailable
>   Progressing    True    NewReplicaSetAvailable
> OldReplicaSets:  <none>
> NewReplicaSet:   deployment-test-5bdd7644cd (3/3 replicas created)
> Events:
>   Type    Reason             Age   From                   Message
>   ----    ------             ----  ----                   -------
>   Normal  ScalingReplicaSet  44s   deployment-controller  Scaled up replica set deployment-test-5bdd7644cd to 3
> ```

As you can see, rather than listing the individual pods, Kubernetes shows us the `ReplicaSet`.
Notice that the name of the `ReplicaSet` is the `Deployment` name and a hash value.

```bash
kubectl get deploy,rs,pod
```

> ```
> NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
> deployment.apps/deployment-test   0/3     3            0           80s
>
> NAME                                         DESIRED   CURRENT   READY   AGE
> replicaset.apps/deployment-test-75d76f9b64   3         3         0       80s
>
> NAME                                   READY   STATUS             RESTARTS   AGE
> pod/deployment-test-75d76f9b64-2msxf   0/1     ImagePullBackOff   0          80s
> pod/deployment-test-75d76f9b64-6rnk5   0/1     ImagePullBackOff   0          80s
> pod/deployment-test-75d76f9b64-gqh9j   0/1     ImagePullBackOff   0          80s
> pod/ds-one-b4nmp                       0/1     CrashLoopBackOff   3          2m1s
> pod/ds-one-b6m4g                       1/1     Running            0          9m11s
> pod/ds-one-lqqqt                       1/1     Running            0          9m11s
> ```

As you can see, the `Deployment` is backed, in this case, by `ReplicaSet` `deployment-test-5bdd7644cd`.
If we go ahead and look at the list of actual Pods.

Try to delete the `ReplicaSet`:

```bash
kubectl delete rs deployment-test-75d76f9b64
```

> ```
> replicaset.apps "deployment-test-75d76f9b64" deleted
> ```

Check the state of Ready POD in ReplicaSet

```bash
kubectl get rs
```

> ```
> NAME                         DESIRED   CURRENT   READY   AGE
> deployment-test-75d76f9b64   3         3         0       47s
> ```

List pods

```bash
kubectl get pods
```

> ```
> NAME                               READY   STATUS    RESTARTS   AGE
> deployment-test-5bdd7644cd-fqcx9   1/1     Running   0          97s
> deployment-test-5bdd7644cd-gf2n8   1/1     Running   0          97s
> deployment-test-5bdd7644cd-tqwwc   1/1     Running   0          97s
> ```

Your pod name have been changed, because the desired state of Deployment create a new ReplicaSet.
Also `ReplicaSet` created 3 replicas of POD.

The same behaviour happen if you try to delete pod:

```bash
kubectl delete pod deployment-test-5bdd7644cd-fqcx9
```

> ```
> pod "deployment-test-5bdd7644cd-fqcx9" deleted
> ```

List pods:

```bash
kubectl get pod
```

> ```
> NAME                               READY   STATUS    RESTARTS   AGE
> deployment-test-5bdd7644cd-bj2qr   1/1     Running   0          15s
> deployment-test-5bdd7644cd-gf2n8   1/1     Running   0          4m20s
> deployment-test-5bdd7644cd-tqwwc   1/1     Running   0          4m20s
> ```

## Scaling Resources

```bash
kubectl scale --replicas=5 deployment deployment-test
```

The new pod have been created in automatically by ReplicaSet according to Deployment desiderated state.

```bash
kubectl set image deployment deployment-test container01=r.deso.tech/library/httpd
```

```bash
kubectl rollout status deployment deployment-test
```

Check your updates:

```bash
kubectl rollout history deployment deployment-test
```

Return to revision 1:

```bash
kubectl rollout undo deployment deployment-test --to-revision=1
```

Let's clean up before we move on.

```bash
kubectl delete deploy deployment-test
```

> ```
> deployment.extensions "deployment-test" deleted
> ```
