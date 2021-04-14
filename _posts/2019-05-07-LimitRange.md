---
layout: post
title: Resource Limits for a Namespace
comments: false
category: blog
---
# Resource Limits for a Namespace


We will create a new namespace and configure another stress deployment to run within. When set stress should not be
able to use the previous amount of resources.

Begin by creating a new namespace called `ns-with-limit` and verify it exists.

## Create Namespace

```bash
kubectl create namespace ns-with-limit
```
> ```
> namespace/ns-with-limit created
> ```

## Create LimitRange

Create a YAML file which limits CPU and memory usage. The kind to use is `LimitRange`.

```bash
vi /home/student/ns-with-limit-range.yaml
```

File: `/home/student/ns-with-limit-range.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: ns-with-limit-range
spec:
  limits:
  - default:
      cpu: 1
      memory: 500Mi
    defaultRequest:
      cpu: 0.5
      memory: 100Mi
    type: Container
```

Create the `LimitRange` object and assign it to the newly created namespace `low-usage-limit`.
You can use `--namespace` or `-n` to declare the namespace.

```bash
kubectl apply --namespace=ns-with-limit -f ns-with-limit-range.yaml
```

> ```
> limitrange/ns-with-limit-range created
> ```

Verify that it works. Remember that every command needs a namespace and context to work.
Defaults are used if not provided.

```bash
kubectl get LimitRange -n ns-with-limit
```

> ```
> NAME                  CREATED AT
> ns-with-limit-range   2019-10-30T09:39:37Z
> ```

Create an unlimited namespace:

```bash
kubectl create ns ns-without-limit
```

> ```
> namespace/ns-without-limit created
> ```

## Create a new deployment in the namespace.

Create a deployment inside namespace `ns-without-limit`

```bash
kubectl -n ns-without-limit create deployment \
	stress-without-limits \
	--image r.deso.tech/whoami/whoami
```

> ```
> deployment.apps/stress-without-limits created
> ```

```bash
kubectl get deploy -n ns-without-limit -o json | jq '.items[].spec.template.spec.containers[]'
```

> ```json
> {
>   "image": "r.deso.tech/whoami/whoami",
>   "imagePullPolicy": "Always",
>   "name": "whoami",
>   "resources": {},
>   "terminationMessagePath": "/dev/termination-log",
>   "terminationMessagePolicy": "File"
> }
> ```

As you can see, no limit was applied to the deployment or the pod:

```bash
kubectl get pod -n ns-without-limit -o json | jq '.items[].spec.containers[].resources'
```

> ```json
> {}
> ```

Create a deployment inside namespace `ns-with-limit`

```bash
kubectl -n ns-with-limit create deployment \
	stress-with-limits   \
	--image r.deso.tech/whoami/whoami
```

> ```
> deployment.apps/stress-with-limits created
> ```


Check the limits on the deployment:

```bash
kubectl get deploy -n ns-with-limit -o json | jq '.items[].spec.template.spec.containers[]'
```

> ```json
> {
>   "image": "r.deso.tech/whoami/whoami",
>   "imagePullPolicy": "Always",
>   "name": "whoami",
>   "resources": {},
>   "terminationMessagePath": "/dev/termination-log",
>   "terminationMessagePolicy": "File"
> }
> ```

Check the limit on the pod:

```bash
kubectl get pod -n ns-with-limit -o json | jq '.items[].spec.containers[].resources'
```
> ```json
> {
>   "limits": {
>     "cpu": "1",
>     "memory": "500Mi"
>   },
>   "requests": {
>     "cpu": "500m",
>     "memory": "100Mi"
>   }
> }
> ```

## Clean up

```bash
kubectl  delete ns ns-without-limit ns-with-limit
```
> ```
> namespace "ns-without-limit" deleted
> namespace "ns-with-limit" deleted
> ```
