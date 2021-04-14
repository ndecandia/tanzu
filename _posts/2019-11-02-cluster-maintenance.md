---
layout: post
title: Cluster Maintenance
comments: false
category: blog
---
# Cluster Maintenance


This lab will cover worker node maintenance as well as cluster backup and restore.
Node maintenance will enable you to take worker nodes offline gracefully and complete maintenance tasks (Kernel
upgrade, Container runtime, hardware changes, etc). Backup and restore will reduce the time to recovery should you lose
the cluster state, it can also help with migrating data from one cluster to another.


Kubectl [`drain`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain) can be used to safely
evict all of your pods from a node before you perform maintenance on the node (e.g. kernel upgrade, hardware
maintenance, etc.). By safely evicting pods with drain the pod’s containers will gracefully terminate and respect any
termination policies you have specified.

Note: By default `kubectl drain` will ignore certain system pods on the node that cannot be killed (Daemonsets, etc);

When `kubectl drain` returns successfully, that indicates that all of the pods (except the ones excluded as described
in the previous paragraph) have been safely evicted. It is then safe to bring down the node by powering down its
physical machine or, if running on a cloud platform, deleting its virtual machine.

Verify current node status.

```bash
kubectl get nodes -o wide
```

> ```
> NAME       STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
> master01   Ready    master   20h     v1.19.6   10.10.95.11   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> master02   Ready    master   14m     v1.19.6   10.10.95.12   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> master03   Ready    master   4m47s   v1.19.6   10.10.95.13   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> worker01   Ready    <none>   19h     v1.19.6   10.10.95.21   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> worker02   Ready    <none>   18h     v1.19.6   10.10.95.22   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> worker03   Ready    <none>   18h     v1.19.6   10.10.95.23   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> ```

Let’s create a sample workload to see how it behaves when the node is drained.

```bash
kubectl create ns lab-drain
```

> ```
> namespace/lab-drain created
> ```

Change default namespace to `lab-drain`:

```bash
kubectl config set-context --current --namespace lab-drain
```

```bash
kubectl create deployment lab-drain --image=nginx
```

> ```
> deployment.apps/lab-drain created
> ```

```bash
kubectl scale deployment lab-drain --replicas=6
```

> ```
> deployment.apps/lab-drain scaled
> ```

Verify that minimum 1 pod has been created on every workers

```bash
kubectl get pods -o wide
```

> ```
> NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
> lab-drain-65f68f47fb-5h562   1/1     Running   0          24s   192.168.39.205   k8s-worker03   <none>           <none>
> lab-drain-65f68f47fb-5ldlh   1/1     Running   0          24s   192.168.79.102   k8s-worker01   <none>           <none>
> lab-drain-65f68f47fb-5zrlp   1/1     Running   0          24s   192.168.69.253   k8s-worker02   <none>           <none>
> lab-drain-65f68f47fb-flfz9   1/1     Running   0          37s   192.168.39.253   k8s-worker03   <none>           <none>
> lab-drain-65f68f47fb-h66xc   1/1     Running   0          24s   192.168.79.77    k8s-worker01   <none>           <none>
> lab-drain-65f68f47fb-hvhpv   1/1     Running   0          24s   192.168.69.220   k8s-worker02   <none>           <none>
> ```

Cordon off `k8s-worker01`

```bash
kubectl cordon worker01
```

> ```
> node/k8s-worker01 cordoned
> ```

Verify `worker1` has been cordoned.

```bash
kubectl get nodes
```

> ```
> NAME           STATUS                     ROLES    AGE    VERSION
> k8s-master01   Ready                      master   239d   v1.19.2
> k8s-master02   Ready                      master   239d   v1.19.2
> k8s-master03   Ready                      master   239d   v1.19.2
> k8s-worker01   Ready,SchedulingDisabled   <none>   95d    v1.19.2
> k8s-worker02   Ready                      <none>   95d    v1.19.2
> k8s-worker03   Ready                      <none>   95d    v1.19.2
> ```

The node should show `SchedulingDisabled`.

Drain the `k8s-worker01` node:

```bash
kubectl drain worker01 --ignore-daemonsets --delete-local-data
```

> ```
> node/k8s-worker01 already cordoned
> evicting pod lab-drain/lab-drain-65f68f47fb-h66xc
> evicting pod lab-drain/lab-drain-65f68f47fb-5ldlh
> ....
> ```

Verify that there are no more pods on `worker1`, and the pod that was there moved to other workers.

```bash
kubectl get pods -o wide
```

> ```
> NAME                         READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
> lab-drain-65f68f47fb-5h562   1/1     Running   0          4m34s   192.168.39.205   k8s-worker03   <none>           <none>
> lab-drain-65f68f47fb-5zrlp   1/1     Running   0          4m34s   192.168.69.253   k8s-worker02   <none>           <none>
> lab-drain-65f68f47fb-cbptw   1/1     Running   0          102s    192.168.69.212   k8s-worker02   <none>           <none>
> lab-drain-65f68f47fb-flfz9   1/1     Running   0          4m47s   192.168.39.253   k8s-worker03   <none>           <none>
> lab-drain-65f68f47fb-hvhpv   1/1     Running   0          4m34s   192.168.69.220   k8s-worker02   <none>           <none>
> lab-drain-65f68f47fb-l229s   1/1     Running   0          102s    192.168.39.192   k8s-worker03   <none>           <none>
> ```

Once that's done, the node would be ready for maintenance.
Let's upgrade OS on `k8s-worker01` to a newer bugfix version.
Open a new terminal tab and connect to `worker1`.

```bash
ssh worker01
sudo apt update && sudo apt-get upgrade -y
```

Switch to the client terminal and verify that `k8s-worker01` is back to ready status.

```bash
kubectl uncordon worker01
```

> ```
> node/worker01 uncordoned
> ```

```bash
kubectl get pods -o wide
```

> ```
> NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
> lab-drain-65f68f47fb-5rdg2   1/1     Running   0          4m4s    172.16.1.154   worker03   <none>           <none>
> lab-drain-65f68f47fb-75ds2   1/1     Running   0          5m33s   172.16.0.214   worker02   <none>           <none>
> lab-drain-65f68f47fb-k82lx   1/1     Running   0          5m33s   172.16.1.152   worker03   <none>           <none>
> lab-drain-65f68f47fb-mvn2l   1/1     Running   0          5m33s   172.16.1.151   worker03   <none>           <none>
> lab-drain-65f68f47fb-pfmwn   1/1     Running   0          5m33s   172.16.0.215   worker02   <none>           <none>
> lab-drain-65f68f47fb-w48gz   1/1     Running   0          4m4s    172.16.0.216   worker02   <none>           <none>
> ```

Existing pods have been moved to other nodes will stay put, if we scale to 1, and after scaling to 6 they will be
distributed to k8s-worker01

```bash
kubectl scale deployment lab-drain --replicas=1
```

> ```
> deployment.apps/lab-drain scaled
> ```

```bash
kubectl scale deployment lab-drain --replicas=6
```

> ```
> deployment.apps/lab-drain scaled
> ```

```bash
kubectl get pods -o wide
```

> ```
> lab-drain-65f68f47fb-9tkp4   1/1     Running   0          14s     172.16.1.155   worker03   <none>           <none>
> lab-drain-65f68f47fb-gxx27   1/1     Running   0          14s     172.16.0.218   worker02   <none>           <none>
> lab-drain-65f68f47fb-lz4jp   1/1     Running   0          14s     172.16.1.156   worker03   <none>           <none>
> lab-drain-65f68f47fb-pfmwn   1/1     Running   0          6m12s   172.16.0.215   worker02   <none>           <none>
> lab-drain-65f68f47fb-pjf5f   1/1     Running   0          14s     172.16.1.19    worker01   <none>           <none>
> lab-drain-65f68f47fb-v8lhf   1/1     Running   0          14s     172.16.1.20    worker01   <none>           <none>
> ```

Clean up our `lab-drain` namespace.

```bash
kubectl delete ns lab-drain
```

> ```
> namespace "lab-drain" deleted
> ```

Return to the `default` namespace

> ```
> kubectl config set-context --current --namespace default
> ```
