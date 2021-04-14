---
layout: post
title: Resource Quota Allocation
comments: false
category: blog
---
# Resource Quota Allocation


Create a new namespace:

```bash
kubectl create ns limited-cpu-memory
```

> ```
> namespace/limited created
> ```

Now create a new file:

```bash
vi /home/student/compute-quota.yaml
```

File: `/home/student/compute-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limited-cpu-memory
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1024Mi
    limits.cpu: "5"
    limits.memory: 2048Mi
```

Create the `compute-quota`:

```bash
kubectl create -f compute-quota.yaml
```
> ```
> resourcequota/compute-quota created
> ```

Check the details of the `compute-quota`:

```bash
kubectl describe quota -n limited-cpu-memory
```
> ```
> Name:            compute-quota
> Namespace:       limited
> Resource         Used  Hard
> --------         ----  ----
> limits.cpu       0     5
> limits.memory    0     2Gi
> requests.cpu     0     1
> requests.memory  0     1Gi
> ```


Similarly, we can define quotas for other Kubernetes objects with the following `ResourceQuota` object.

Create a new namespace:

```bash
kubectl create ns limited-count-object
```
> ```
> namespace/limited-count-object created
> ```

Create a new file:

```bash
vi object-quota.yaml
```

File: `/home/student/object-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: limited-count-object
spec:
  hard:
    configmaps: "5"
    persistentvolumeclaims: "4"
    pods: "10"
    secrets: "10"
    services: "10"
    services.nodeports: "2"
```

Apply these object quotas:

```bash
kubectl create -f object-quota.yaml
```
> ```
> resourcequota/object-quota created
> ```

Check on the `object-counts` object status:

```bash
kubectl describe quota object-counts -n limited-count-object
```
> ```
> Name:                   object-counts
> Namespace:              limited-count-object
> Resource                Used  Hard
> --------                ----  ----
> configmaps              0     5
> persistentvolumeclaims  0     4
> pods                    0     10
> secrets                 1     10
> services                0     10
> services.nodeports      0     2
> ```

At this point, try to create a simple deployment:

```bash
kubectl create deployment test01 --image=r.deso.tech/library/nginx -n limited-count-object
```
> ```
> deployment.apps/test01 created
> ```

Scale your `test01` deployment to 11 replicas:

```bash
kubectl scale deployment test01 --replicas=11 -n limited-count-object
```
> ```
> deployment.apps/test01 scaled
> ```

Now, check your environment and investigate what happened:

```bash
kubectl get deploy,rs,pod -n limited-count-object
```
> ```
> NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
> deployment.apps/test01   10/11   10           10          7m13s
>
> NAME                                DESIRED   CURRENT   READY   AGE
> replicaset.apps/test01-67f6859667   11        10        10      40s
>
> NAME                          READY   STATUS    RESTARTS   AGE
> pod/test01-67f6859667-44rdg   1/1     Running   0          40s
> pod/test01-67f6859667-6jf2j   1/1     Running   0          40s
> pod/test01-67f6859667-f52d8   1/1     Running   0          40s
> pod/test01-67f6859667-gfktj   1/1     Running   0          40s
> pod/test01-67f6859667-h7nrh   1/1     Running   0          40s
> pod/test01-67f6859667-krgjj   1/1     Running   0          39s
> pod/test01-67f6859667-lhxdk   1/1     Running   0          40s
> pod/test01-67f6859667-lxvgz   1/1     Running   0          40s
> pod/test01-67f6859667-thpwd   1/1     Running   0          39s
> pod/test01-67f6859667-vw5n5   1/1     Running   0          39s
> ```

Where are your 11 pods?
Investigate your deployment and replicaset:

```bash
kubectl describe deploy test01 -n limited-count-object
```

```bash
kubectl describe rs test01-xxxxxx -n limited-count-object
```

If you look in the events, you'll see some errors.
Kubernetes can't schedule all your pods because you reached the maximum defined in `ResourseQuota`.

Check on the `object-counts` object status:

```bash
kubectl describe quota object-counts -n limited-count-object
```
> ```
> Name:                   object-counts
> Namespace:              limited-count-object
> Resource                Used  Hard
> --------                ----  ----
> configmaps              0     5
> persistentvolumeclaims  0     4
> pods                    10    10
> secrets                 1     10
> services                0     10
> services.nodeports      0     2
> ```

## Clean up

```bash
kubectl  delete ns limited-cpu-memory limited-count-object
```
> ```
> namespace "limited-cpu-memory" deleted
> namespace "limited-count-object" deleted
> ```
