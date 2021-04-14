---
layout: post
title: Check your cluster
comments: false
category: blog
---
# Check your cluster


Additional connection and service information information can be collected using the `cluster-info` command:

```bash
kubectl cluster-info
```
> ```
> Kubernetes master is running at https://10.10.98.10:6443
> KubeDNS is running at https://10.10.98.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
>
> To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
> ```

Here, the output is showing the endpoint for our Kubernetes master as well as the KubeDNS service endpoint.

To see information about each of the individual nodes that are members of your cluster with a wide output, use the `get
nodes` command:

```bash
kubectl get nodes -o wide
```

> ```
> NAME       STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
> master01   Ready    master   98m     v1.19.6   10.10.95.11   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> worker01   Ready    <none>   21m     v1.19.6   10.10.95.21   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> worker02   Ready    <none>   9m39s   v1.19.6   10.10.95.22   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> worker03   Ready    <none>   8m14s   v1.19.6   10.10.95.23   <none>        Ubuntu 20.04.1 LTS   5.4.0-48-generic   docker://19.3.14
> ```

This lists the status, roles, connection information, and version numbers of the core software running on each of the
nodes. If you need to perform maintenance on your cluster nodes or log in to debug an issue, this command can help
provide the information you need.

## Viewing Resource and Event Information

To get an overview of the namespaces available within a cluster, use the get namespaces command:

```bash
kubectl get namespaces
```
> ```
> NAME              STATUS   AGE
> calico-system     Active   67m
> default           Active   99m
> kube-node-lease   Active   99m
> kube-public       Active   99m
> kube-system       Active   99m
> tigera-operator   Active   75m
> ```

This shows the namespace partitions defined within the current cluster.

To get an overview of all of the resources running on your cluster, across all namespaces, issue the following command:

```bash
kubectl get all --all-namespaces
```

> ```
> <output_omitted>
> ```

The output displays the namespace each resource is deployed in as well as the resource name prefixed by the resource
type (output_omitted in the examples shown above). Afterwards, information about the ready and running statuses of each
resource helps determine if the processes are operating healthily.

To view the events associated with your resources, use the `get events` command:

```bash
kubectl get events --all-namespaces
```

> ```
> <output_omitted>
> ```

This lists the most recent events logged by your resources, including the event message and the reason it was
triggered.
