---
layout: post
title: Portforward and Proxy
comments: false
category: blog
---
# Portforward and Proxy


## Forward a local port to a port on the Pod.

Start a whoami application by entering this command:

```bash
kubectl create deployment whoami --image=r.deso.tech/whoami/whoami --port=80
```

> ```
> deployment.apps/whoami created
> ```

`kubectl port-forward` allows using resource name, such as a pod name, to select a matching pod to port forward to.

On the first terminal:

```bash
kubectl port-forward deployment/whoami 9000:80
```

> ```
> Forwarding from 127.0.0.1:9000 -> 80
> Forwarding from [::1]:9000 -> 80
> ```

Now open a second terminal on a student desktop and run `curl`:

```bash
curl localhost:9000
```

> ```
> +-------------------------+
> |        HOSTNAME         |
> +-------------------------+
> | whoami-5d6d99c644-b2vl5 |
> +-------------------------+
> +-------------------+-----------+
> |        IP         | INTERFACE |
> +-------------------+-----------+
> | 127.0.0.1/8       | lo        |
> | 192.168.39.194/32 | eth0      |
> +-------------------+-----------+
> +-------------------------+
> |         REQUEST         |
> +-------------------------+
> | GET / HTTP/1.1 Host:    |
> | localhost User-Agent:   |
> | curl/7.64.1 Accept: */* |
> |                         |
> +-------------------------+
> ```

Now delete the deployment:

```bash
kubectl delete deploy whoami
```

> ```
> deployment.apps "whoami" deleted
> ```

## Using kubectl to start a proxy server

You can also use a proxy server to start a proxy to the Kubernetes API server:

From student desktop:

```bash
kubectl proxy --port=8080
```

> ```
> Starting to serve on 127.0.0.1:8080
> ```

Do not close this terminal and open another terminal console.
Now you can try `curl` and get responses from Kubernetes' API server directly on your student desktop:

```bash
curl localhost:8080
```

> ```
> {
>   "paths": [
>     "/api",
>     "/api/v1",
>     "/apis",
>     "/apis/",
> . . .
> }
> ```
