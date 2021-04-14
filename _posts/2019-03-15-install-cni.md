---
layout: post
title: Install CNI
comments: false
category: blog
---
# Install CNI


From the master reading the output of kubeadm installation:

```bash
cat /home/student/kubeadm-init.out
```

> ```
> <output_omitted>

> You should now deploy a pod network to the cluster.
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>   https://kubernetes.io/docs/concepts/cluster-administration/addons/

> <output_omitted>
> ```

Deciding which pod network to use for Container Networking Interface (CNI) should take into account the expected
demands on the cluster. There can be only one pod network per cluster, although the CNI-Genie project (Huawei) is
trying to change this.

The network must allow container-to-container, pod-to-pod, pod-to-service, and external-to-service communications. As
Docker uses host-private networking, using the `docker0` virtual bridge and `veth` interfaces would require being on
that host to communicate.

We will use Calico as a network plugin which will allow us to use Network Policies later in the course. Currently
Calico does not deploy using CNI by default. The 3.9 version of Calico has one configuration file for flexibility with
RBAC. Download the configuration files for. You can apply configuration files directly from Project Calico's manifests.

For more information about using Calico, see [Quickstart for Calico on
Kubernetes](https://docs.projectcalico.org/latest/getting-started/kubernetes/), Installing Calico for policy and
networking, and other related resources.

Apply the network plugin configuration to your cluster. Remember to copy the file to the current, non-root user
directory first.

Exit and come back to the student console:

```bash
ssh student@student
```

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

> ```
> namespace/tigera-operator created
> podsecuritypolicy.policy/tigera-operator created
> serviceaccount/tigera-operator created
> customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
> customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
> customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
> clusterrole.rbac.authorization.k8s.io/tigera-operator created
> clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
> deployment.apps/tigera-operator created
> ```

Download the `custom-resources.yaml` file

```bash
wget https://docs.projectcalico.org/manifests/custom-resources.yaml
```
you can configure it with a specific podCIDR:

```bash
vi custom-resources.yaml
```

> ```yaml
> # This section includes base Calico installation configuration.
> # For more information, see: https://docs.projectcalico.org/v3.16/reference/installation/api#operator.tigera.io/v1.Installation
> apiVersion: operator.tigera.io/v1
> kind: Installation
> metadata:
>   name: default
> spec:
>   # Configures Calico networking.
>  calicoNetwork:
>     # Note: The ipPools section cannot be modified post-install.
>     ipPools:
>     - blockSize: 26
>       cidr: 192.168.0.0/16
>       encapsulation: VXLANCrossSubnet
>       natOutgoing: Enabled
>       nodeSelector: all()
> ```

Edit the cidr with the same IP assigned inside the `ClusterConfiguration.yaml`.

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: r.deso.tech
  imagePath: calico
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 172.16.0.0/23
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

Save it and apply:

```bash
kubectl apply -f custom-resources.yaml
```

> ```
> installation.operator.tigera.io/default created
> ```

Check what yaml created for us

Calico have created a pod as daemonSet in every node, now we have only one node, so you can read 1

```bash
kubectl get ds --namespace calico-system
```

> ```
>  NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
> calico-node   1         1         1       1            1           beta.kubernetes.io/os=linux   7m59s
> ```

This is the pods created:

```bash
kubectl get pods --namespace calico-system | grep calico
```

> ```
> calico-kube-controllers-59f54d6bbc-tg697   1/1     Running   0          5m24s
> calico-node-qv4hn                          1/1     Running   0          5m24s
> ```

View the available nodes of the cluster. It can take a minute or two for the status to change from Not Ready to Ready.

```bash
kubectl get nodes
```

> ```
> NAME     STATUS   ROLES    AGE   VERSION
> master   Ready    master   82m   v1.19.5
> ```
