---
layout: post
title: Network Policies
comments: false
category: blog
---
# Network Policies


## Creating NetworkPolicies

So far, we've seen pods communicating openly with each other, but we may want the ability to restrict traffic between
pods. For this, there are *networkPolicies*.

1. Create a big yaml specification for deploy an origin and a destination deployment.
   Also you will add a service for reach the destination:

   ```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: origin
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: origin
  template:
    metadata:
      labels:
        app: origin
    spec:
      containers:
      - name: origin-container
        image: r.deso.tech/library/netshoot
        command: ["sleep"]
        args: ["1000000"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: destination
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: destination
  template:
    metadata:
      labels:
        app: destination
    spec:
      containers:
      - name: destination-container
        image: r.deso.tech/whoami/whoami
---
apiVersion: v1
kind: Service
metadata:
  name: svc-destination
  namespace: default
spec:
  selector:
    app: destination
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f stack.yaml
```

> ```
> deployment.apps/origin created
> deployment.apps/destination created
> service/svc-destination created
> ```

2. Create a networkPolicy that matches your `destination` service pods by label selector, and doesn't allow any
   ingress:

   ```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: destination
```

```bash
kubectl apply -f network-policy.yaml
```

> ```
> networkpolicy.networking.k8s.io/test-network-policy created
> ```

With no `ingress` rules defined, *all* ingress to matching pods will be blocked.

3. Exec a bash shell in your `origin` pod as you did before, and attempt to `curl` a `destination` pod via `curl
   svc-destination:8080`; the curl will hang, blocked by the networkPolicy which by default blocks all ingress to pods
   matched by its label selector.

```bash
kubectl get pod
```

> ```
> NAME                           READY   STATUS    RESTARTS   AGE
> destination-6f66576498-25qpv   1/1     Running   0          102s
> destination-6f66576498-clzkr   1/1     Running   0          102s
> destination-6f66576498-mq6pk   1/1     Running   0          102s
> origin-5b945ff9c5-5fkd9        1/1     Running   0          102s
> ```

```bash
kubectl exec -it origin-5b945ff9c5-5fkd9 -- /bin/bash
```

> ```
> bash-5.0#
> ```

```bash
curl svc-destination
```

> ```
> ^C
> ```

```bash
exit
```

> ```
> exit
> command terminated with exit code 130
> ```

4. Delete your networkPolicy.

```bash
kubectl delete networkpolicies test-network-policy
```

> ```
> networkpolicy.extensions "test-network-policy" deleted
> ```

5. Create a new networkPolicy, similar to the last one but this time allowing ingress from pods labeled as `app:
   origin`:

   ```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: good-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: destination
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: origin
```

```bash
kubectl apply -f good-networkpolicy.yaml
```

> ```
> networkpolicy.networking.k8s.io/good-network-policy created
> ```

Try once again to `curl` your `destination` pods from your `origin` pod; you'll once again be successful, as our new
networkPolicy defines an ingress rule that allows ingress from pods with the `app: origin` label.

```bash
kubectl exec -it origin-5b945ff9c5-5fkd9 -- /bin/bash
```

> ```
> bash-5.0#
> ```

```bash
curl svc-destination
```

> ```
> Hostname: destination-6f66576498-clzkr
> IP: 127.0.0.1
> IP: 192.168.32.195
> GET / HTTP/1.1
> Host: svc-destination
> User-Agent: curl/7.65.1
> Accept: */*
> ```

```bash
exit
```

> ```
> exit
> ```

6. Delete the networkPolicy you created in this section.

```bash
kubectl delete networkpolicies good-network-policy
```

> ```
> networkpolicy.extensions "good-network-policy" deleted
> ```

```bash
kubectl delete deployments destination origin
```

> ```
> deployment.extensions "destination" deleted
> deployment.extensions "origin" deleted
> ```
