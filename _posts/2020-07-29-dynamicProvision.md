---
layout: post
title: Dynamic Storage Provisioning
comments: false
category: blog
---
# Dynamic Storage Provisioning


In our environment, we have a cluster with storage provisioner configured.
Now, we need to create a PVC object and consume it in a pod.

Let's create a PVC.

First of all, create a folder for this lab:

```bash
mkdir ~/storage ; cd ~/storage
```

The following description of a persistentVolumeClaim (PVC) uses our `storageClass` to request 1GB of storage that can
be used to populate a volume in a pod. Place it in a file `pvc01.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  storageClassName: <your_sc>
```

### Create the PVC:

```bash
kubectl apply -f pvc01.yaml
```

> ```
> persistentvolumeclaim/pvc01 created
> ```

```bash
kubectl get pvc pvc01
```

The following yaml describes a pod which uses the above PVC to populate a volume, mounts that volume into a container,
and periodically writes a message to that storage location so we can see it in action.
Place this in a file named `pod-pvc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pvc-pod
spec:
  volumes:
  - name: volume01
    persistentVolumeClaim:
      claimName: pvc01
  containers:
  - name: hello-container
    image: busybox
    command:
      - sh
      - -c
      - 'while true; do echo "`date` [`hostname`] Hello from the dynamically provisioned \
          Persistent Volume." >> /mnt/store/greet.txt; sleep $(($RANDOM % 5 + 300)); done'
    volumeMounts:
    - mountPath: /mnt/store
      name: volume01
```

### Deploy this pod:

```bash
kubectl apply -f pod-pvc.yaml
```

> ```
> pod/hello-pvc-pod created
> ```

```bash
kubectl get pod hello-pvc-pod
```

> ```
> NAME                       READY   STATUS    RESTARTS   AGE
> hello-pvc-pod              1/1     Running   0          11s
> ```

### Verify data is being written within the container’s filesystem where we mounted the PV:

```bash
kubectl exec hello-pvc-pod  -- cat /mnt/store/greet.txt
```

### Verify that the container is writing to the local persistent volume:

```bash
kubectl describe pod hello-pvc-pod
```

> ```
> Name:         hello-pvc-pod
> Namespace:    default
> Priority:     0
> Node:         node1/10.10.65.94
> ....
> Volumes:
>   volume01:
>     Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
>     ClaimName:  pvc01
>     ReadOnly:   false
> ....
> ```

### Check the details of the PVC:

```bash
kubectl get pvc pvc01
```

> ```
> NAME                 STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS     AGE
> pvc01   Bound    pvc-xxx   1G         RWO            local-hostpath   10m
> ```

A new persistent volume named `pvc-xxx` has been dynamically provisioned by our storage class.
Let’s look at the details of the persistent volume.

### Check the details of the persistent volume:

```bash
kubectl describe pv <Volume name from above>
```

> ```
> Name:           pvc-xxx
> ....
> Node Affinity:
>   Required Terms:
>     Term 0:        kubernetes.io/hostname in [node1]
> ....
> Source:
>     Type:  LocalVolume (a persistent volume backed by local storage on a node)
>     Path:  /data/pvc-xxx
> ```

Note in the output the name of the node `node1` and actual location within the node’s filesystem `/data/pvc-xxx`.
We can confirm that the `greet.txt` file is accessible from the source path.

### Confirm the `greet.txt` file location, where `nodeX` is whichever node you found your pod and volume scheduled on:

```bash
cat <PV host path found above>/greet.txt
```

> ```
> Fri May 29 20:02:55 UTC 2020 [master01] Hello from the dynamically provisioned Persistent Volume.
> Fri May 29 20:02:55 UTC 2020 [master01] Hello from the dynamically provisioned Persistent Volume.
> ```

### Clean Up

Delete the pod.

```bash
kubectl delete pod hello-pod-pvc
```

Delete the pvc.

```bash
kubectl delete pvc pvc01
```
