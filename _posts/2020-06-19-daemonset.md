---
layout: post
title: DaemonSet Update
comments: false
category: blog
---
# DaemonSet Update


## Create a Daemonet

First of all, we need to create a daemonset. Create the following file:

<!--codeinclude-->
[ds.yaml](assets/daemonset/ds.yaml)
<!--/codeinclude-->

[Download `ds.yaml` here](assets/daemonset/ds.yaml).

Apply the file:

```bash
kubectl apply -f ds.yaml
```

> ```
> daemonset/ds-one created
> ```

## Update a DaemonSet (OnDelete)

Kubernetes supports two strategies for upgrading daemonsets, let's find the default:

```bash
kubectl get ds ds-one -o yaml \
| grep -A 3 Strategy
```

Edit the object to use the `OnDelete` update strategy. This would allow the manual termination of some of the pods,
resulting in an updated image when they are recreated.

```bash
kubectl edit ds ds-one
```

> ```
> ....
> updateStrategy:
>   rollingUpdate:
>     maxUnavailable: 1
>   type: OnDelete # <-- 'RollingUpdate' should be replaced with 'OnDelete'
> ....
> ```

Update the `DaemonSet` to use a newer version of the nginx server.
This time use the set command instead of edit.
Set the version to be 1.16.1-alpine.

```bash
kubectl set image ds ds-one nginx=httpd
```

> ```
> daemonset.apps/ds-one image updated
> ```

Verify that the `Image:` parameter for the Pod checked in the previous section is unchanged.

```bash
kubectl describe po ds-one-b2dcv | grep Image:
```

> ```
>     Image:          r.deso.tech/library/nginx
> ```

Delete a Pod. Wait until the replacement Pod is running and check the version.

```bash
kubectl delete po ds-one-b2dcv
```

> ```
> pod "ds-one-b2dcv" deleted
> ```

```bash
kubectl get pod
```

> ```
> NAME READY STATUS RESTARTS AGE
> ds-one-xc86w 1/1 Running 0 21s
> ds-one-z31r4 1/1 Running 0 4m12s
> ```

```bash
kubectl describe pod ds-one-xc86w | grep Image:
```

> ```
>     Image:          r.deso.tech/library/httpd
> ```

View the image running on the older pods. It should still show nginx.

```bash
kubectl describe pod ds-one-z31r4 | grep Image:
```

> ```
>     Image:          r.deso.tech/library/nginx
> ```

View the history of changes for the `DaemonSet`. You should see two revisions listed.
As we did not use the `--record` option we didn’t see why the object updated.

```bash
kubectl rollout history ds ds-one
```

> ```
> REVISION CHANGE-CAUSE
> 1        <none>
> 2        <none>
> ```

View the settings for the various versions of the `DaemonSet`.
The `Image:` line should be the only difference between the two outputs.

```bash
kubectl rollout history ds ds-one --revision=1
```

> ```
> Pod Template:
> Labels: system=DaemonSetOne
> Containers:
> nginx:
> Image: r.deso.tech/library/nginx
> Port: 80/TCP
> Environment: <none>
> Mounts: <none>
> Volumes: <none>
> ```

```bash
kubectl rollout history ds ds-one --revision=2
```

> ```
> ....
> Image: r.deso.tech/library/httpd
> .....
> ```

Use `kubectl rollout undo` to change the `DaemonSet` back to an earlier version.
As we are still using the `OnDelete` strategy there should be no change to the Pods.

```bash
kubectl rollout undo ds ds-one --to-revision=1
```

> ```
> daemonset.apps/ds-one rolled back
> ```

```bash
kubectl describe pod ds-one-xc86w | grep Image:
```

> ```
> Image: r.deso.tech/library/nginx
> ```

Delete the Pod, wait for the replacement to spawn then check the image version again.

```bash
kubectl delete pod ds-one-xc86w
```

> ```
> pod "ds-one-xc86w" deleted
> ```

```bash
kubectl get pod
```

> ```
> NAME READY STATUS RESTARTS AGE
> ds-one-qc72k 1/1 Running 0 10s
> ds-one-xc86w 0/1 Terminating 0 12m
> ds-one-z31r4 1/1 Running 0 28m
> ```

```bash
kubectl describe po ds-one-qc72k | grep Image:
```

> ```
> Image: r.deso.tech/library/nginx
> ```

View the details of the `DaemonSet`. The Image should be v1.15.1 in the output.

```bash
kubectl describe ds | grep Image:
```

> ```
> Image: r.deso.tech/library/nginx
> ```

View the current configuration for the `DaemonSet` in YAML output.
Look for the updateStrategy:

```bash
kubectl get ds ds-one -o yaml
```

> ```
> apiVersion: apps/v1
> kind: DaemonSet
> .....
> terminationGracePeriodSeconds: 30
> updateStrategy:
> type: OnDelete
> status:
> currentNumberScheduled: 2
> .....
> ```

## Update a DaemonSet (RollingUpdate)

Create a new `DaemonSet`, this time setting the update policy to `RollingUpdate`.
Begin by generating a new config file.

```bash
kubectl get ds ds-one -o yaml > ds2.yaml
```

Edit the file. Change the name, around line 70 and the update strategy around line 101, back to the default
`RollingUpdate`.

```bash
vim ds2.yaml
```

> ```
> ....
> name: ds-two
> ....
> type: RollingUpdate
> ```

Create the new `DaemonSet` and verify the `nginx` version in the new pods.

```bash
kubectl create -f ds2.yaml
```

> ```
> daemonset.apps/ds-two created
> ```

```bash
kubectl get pod
```

> ```
> NAME READY STATUS RESTARTS AGE
> ds-one-qc72k 1/1 Running 0 28m
> ds-one-z31r4 1/1 Running 0 57m
> ds-two-10khc 1/1 Running 0 5m
> ds-two-kzp9g 1/1 Running 0 5m
> ```

```bash
kubectl describe po ds-two-10khc | grep Image:
```

> ```
> Image: r.deso.tech/library/nginx
> ```

Edit the configuration file and set the image to a newer version such as 1.16.1-alpine. Include the `--record` option.

```bash
kubectl edit ds ds-two --record
```

> ```
> ....
> - image: r.deso.tech/library/httpd
> .....
>
> ```

View the age of the `DaemonSet`s. It should be around ten minutes old, depending on how fast you type.

```bash
kubectl get ds ds-two
```

> ```
> NAME DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE-SELECTOR AGE
> ds-two 2 2 2 2 2 <none> 10m
> ```

Now view the age of the Pods. Two should be much younger than the `DaemonSet`. They are also a few seconds apart
due to the nature of the rolling update where one then the other pod was terminated and recreated.

```bash
kubectl get pod
```

> ```
> NAME        READY STATUS RESTARTS AGE
> ds-one-qc72k 1/1 Running 0        36m
> ds-one-z31r4 1/1 Running 0        1h
> ds-two-2p8vz 1/1 Running 0        34s
> ds-two-8lx7k 1/1 Running 0        32s
> ```

Verify the Pods are using the new version of the software.

```bash
kubectl describe po ds-two-8lx7k | grep Image:
```

> ```
> Image: r.deso.tech/library/httpd
> ```

View the rollout status and the history of the `DaemonSet`s.

```bash
kubectl rollout status ds ds-two
```

> ```
> daemon set "ds-two" successfully rolled out
> ```

```bash
kubectl rollout history ds ds-two
```

> ```
> REVISION CHANGE-CAUSE
> 1        <none>
> 2        kubectl edit ds ds-two --record=true
> ```

View the changes in the update they should look the same as the previous history, but did not require the Pods to be
deleted for the update to take place.

```bash
kubectl rollout history ds ds-two --revision=2
```

> ```
> ...
> Image: r.deso.tech/library/httpd
> ```

Clean up the environment by removing the `DaemonSet`s.

```bash
kubectl delete ds ds-one ds-two
```

> ```
> daemonset.apps "ds-one" deleted
> daemonset.apps "ds-two" deleted
> ```
