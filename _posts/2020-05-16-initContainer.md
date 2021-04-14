---
layout: post
title: Pod InitContainer
comments: false
category: blog
---
# Pod InitContainer


Create 2 service, so we can emulate a `clusterIP` service.

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: webservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
kind: Service
apiVersion: v1
metadata:
  name: mysql
spec:
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```

Apply the file:

```bash
kubectl apply -f initcontainer-svc.yaml
```

> ```
> service/webservice created
> service/mysql created
> ```

This services will not have any pod attached, the `clusterIP` still be resolved.

```bash
kubectl get svc
```

> ```
> NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
> mysql        ClusterIP   10.106.152.72   <none>        3306/TCP   24s
> webservice   ClusterIP   10.97.16.97     <none>        80/TCP     24s
> ```

Create a Pod with two InitContainers that wait until the service answer:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: initcontainer-1
  labels:
    app: initcontainer-1
spec:
  containers:
  - name: container-1
    image: r.deso.tech/library/centos
    command: ['bash', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-container1
    image: r.deso.tech/library/netshoot
    command: ['sh', '-c', 'until nslookup webservice; do echo waiting for webservice application; sleep 2; done;']
  - name: init-container2
    image: r.deso.tech/library/netshoot
    command: ['sh', '-c', 'until nslookup mysql; do echo waiting for mysql database; sleep 2; done;']
```

Apply the yaml file:

```bash
kubectl apply -f initcontainer-1.yaml
```
Wait until the pod will start:

```bash
kubectl get pod
```

> ```
> NAME                           READY   STATUS    RESTARTS   AGE
> initcontainer-1                1/1     Running   0          9s
> ```

```bash
kubectl logs initcontainer-1
```

> ```
> The app is running!
> ```

You should see "The app is running!", because pass all the initcontainer without any problem.

Now delete the pod:

```bash
kubectl delete pod initcontainer-1 --force --grace-period=0
```

> ```
> warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
> pod "initcontainer-1" force deleted
> ```

You are going to delete one of the services:

```bash
kubectl delete svc mysql
```

> ```
> service "mysql" deleted
> ```

Now apply again the yaml file:

```bash
kubectl apply -f initcontainer-1.yaml
```

Immediately run:

```bash
kubectl get pod -w
```

> ```
> NAME                           READY   STATUS     RESTARTS   AGE
> initcontainer-1                0/1     Init:1/2   0          4s
> initcontainer-1                0/1     Init:1/2   0          6s
> ```

You will see that the pod will stay indefinitely waiting that `nslookup` resolves it.
After 1 minute the watching command will exit, but the pod still to wait the `initContainer`.

Now try to `apply` again the file of the service mysql.
This operation will permit to resolve again the load balancer svc mysql and the `initContainer` will continue the step:

```bash
kubectl get pod
```

> ```
> NAME                           READY   STATUS     RESTARTS   AGE
> initcontainer-1                0/1     Init:1/2   0          4s
> initcontainer-1                0/1     Init:1/2   0          6s
> ```

```bash
kubectl apply -f initcontainer-svc.yaml
```

> ```
> service/webservice unchanged
> service/mysql created
> ```

Check again the pod and you will see the Pod in running state.

```bash
kubectl get pod -w
```

> ```
> NAME              READY   STATUS            RESTARTS   AGE
> initcontainer-1   0/1     PodInitializing   0          4m26s
> initcontainer-1   1/1     Running           0          4m27s
> ```

Delete the pod:

```bash
kubectl delete pod initcontainer-1 --force --grace-period=0
```

> ```
> warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
> pod "initcontainer-1" force deleted
> ```

Delete the services:

```bash
kubectl delete svc mysql webservice
```

> ```
> service "mysql" deleted
> service "webservice" deleted
> ```
