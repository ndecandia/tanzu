---
layout: post
title: Upgrade Cluster
comments: false
category: blog
---
# Upgrade Cluster


```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES    AGE   VERSION
> master01   Ready    master   20h   v1.19.6
> master02   Ready    master   25m   v1.19.6
> master03   Ready    master   15m   v1.19.6
> worker01   Ready    <none>   19h   v1.19.6
> worker02   Ready    <none>   18h   v1.19.6
> worker03   Ready    <none>   18h   v1.19.6
> ```

## Upgrade the first control plane node

In a new terminal tab, run:

```bash
ssh master01
```

```bash
sudo apt update
```

```bash
sudo apt-cache madison kubeadm
```

> ```
> kubeadm |  1.20.1-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.20.0-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.19.6-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.19.5-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.19.4-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.19.3-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.19.2-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> kubeadm |  1.19.1-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
> ......
> ```

On your first control plane node, upgrade `kubeadm`:

> ```
> sudo apt-mark unhold kubeadm && \
> sudo apt-get update && sudo apt-get install -y kubeadm=1.20.1-00 && \
> sudo apt-mark hold kubeadm
> ```

Verify that the download works and has the expected version:

```bash
kubeadm version
```

> ```
> kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.1" .....
> ```

On the `student` machine:

```bash
kubectl drain master01 --ignore-daemonsets
```

```
node/master01 cordoned
WARNING: ignoring DaemonSet-managed Pods: calico-system/calico-node-chc22, kube-system/kube-proxy-x2lts, metallb-system/speaker-zmgzn, monitoring/node-exporter-tjmxf
evicting pod calico-system/calico-typha-7c6d988549-nqn5r
evicting pod tigera-operator/tigera-operator-657cc89589-hcs8m
evicting pod calico-system/calico-kube-controllers-546d44f5b7-d6nlx
evicting pod kube-system/coredns-f9fd979d6-422m2
evicting pod kube-system/coredns-f9fd979d6-m6g9f
pod/calico-typha-7c6d988549-nqn5r evicted
pod/tigera-operator-657cc89589-hcs8m evicted
pod/calico-kube-controllers-546d44f5b7-d6nlx evicted
pod/coredns-f9fd979d6-422m2 evicted
pod/coredns-f9fd979d6-m6g9f evicted
node/master01 evicted
```

On the control plane node, `master01` run:

```bash
sudo kubeadm upgrade plan
```

> ```
> [upgrade/config] Making sure the configuration is correct:
> [upgrade/config] Reading configuration from the cluster...
> [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
> [preflight] Running pre-flight checks.
> [upgrade] Running cluster health checks
> [upgrade] Fetching available versions to upgrade to
> [upgrade/versions] Cluster version: v1.19.6
> [upgrade/versions] kubeadm version: v1.20.1
> [upgrade/versions] Latest stable version: v1.20.1
> [upgrade/versions] Latest stable version: v1.20.1
> [upgrade/versions] Latest version in the v1.19 series: v1.19.6
> [upgrade/versions] Latest version in the v1.19 series: v1.19.6
>
> Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
> COMPONENT   CURRENT       AVAILABLE
> kubelet     6 x v1.19.6   v1.20.1
>
> Upgrade to the latest stable version:
>
> COMPONENT                 CURRENT    AVAILABLE
> kube-apiserver            v1.19.6    v1.20.1
> kube-controller-manager   v1.19.6    v1.20.1
> kube-scheduler            v1.19.6    v1.20.1
> kube-proxy                v1.19.6    v1.20.1
> CoreDNS                   1.7.0      1.7.0
> etcd                      3.4.13-0   3.4.13-0
>
> You can now apply the upgrade by executing the following command:
>
> 	kubeadm upgrade apply v1.20.1
>
> _____________________________________________________________________
>
>
> The table below shows the current state of component configs as understood by this version of kubeadm.
> Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
> resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
> upgrade to is denoted in the "PREFERRED VERSION" column.
>
> API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
> kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
> kubelet.config.k8s.io     v1beta1           v1beta1             no
> _____________________________________________________________________
>
> ```

This command checks that your cluster can be upgraded, and fetches the versions you can upgrade to.
It also shows a table with the component config version states.

Choose a version to upgrade to, and run the appropriate command. For example:

```bash
sudo kubeadm upgrade apply v1.20.1
```

> ```
> [upgrade/config] Making sure the configuration is correct:
> [upgrade/config] Reading configuration from the cluster...
> [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
> [preflight] Running pre-flight checks.
> [upgrade] Running cluster health checks
> [upgrade/version] You have chosen to change the cluster version to "v1.20.1"
> [upgrade/versions] Cluster version: v1.19.6
> [upgrade/versions] kubeadm version: v1.20.1
> [upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
> [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
> [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
> [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
>
> .........<omitted>
>
>
> [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.20.1". Enjoy!
>
> [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
> ```

Manually upgrade your CNI provider plugin.

Your Container Network Interface (CNI) provider may have its own upgrade instructions to follow.

This step is not required on additional control plane nodes if the CNI provider runs as a DaemonSet.

```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet=1.20.1-00 kubectl=1.20.1-00 && \
sudo apt-mark hold kubelet kubectl
```

While you are performing the upgrade che the node status on student terminal

```bash
kubectl get nodes
```

> ```
> master01   Ready,SchedulingDisabled   control-plane,master   20h   v1.20.1
> master02   Ready                      control-plane,master   38m   v1.19.6
> master03   Ready                      control-plane,master   28m   v1.19.6
> worker01   Ready                      <none>                 19h   v1.19.6
> worker02   Ready                      <none>                 19h   v1.19.6
> worker03   Ready                      <none>                 19h   v1.19.6
> ```

On the `master01` terminal
Restart the kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Return on the `student` terminal and uncordon `master01`

```bash
kubectl uncordon master01
```

> ```
> node/master01 uncordoned
> ```

```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES                  AGE   VERSION
> master01   Ready    control-plane,master   20h   v1.20.1      <----------
> master02   Ready    control-plane,master   40m   v1.19.6
> master03   Ready    control-plane,master   30m   v1.19.6
> worker01   Ready    <none>                 19h   v1.19.6
> worker02   Ready    <none>                 19h   v1.19.6
> worker03   Ready    <none>                 19h   v1.19.6
> ```

## Upgrade additional control plane,

### Repeat the previous steps on master02 and master03


**The final result will be like this:**

> ```
> NAME       STATUS   ROLES                  AGE   VERSION
> master01   Ready    control-plane,master   20h   v1.20.1
> master02   Ready    control-plane,master   40m   v1.20.1
> master03   Ready    control-plane,master   30m   v1.20.1
> worker01   Ready    <none>                 19h   v1.19.6
> worker02   Ready    <none>                 19h   v1.19.6
> worker03   Ready    <none>                 19h   v1.19.6
> ```

The upgrade procedure on worker nodes should be executed one node at a time or few nodes at a time,
without compromising the minimum required capacity for running your workloads.


## Upgrade kubeadm

Upgrade `kubeadm` on all worker nodes:


### Drain the first worker node

```bash
kubectl drain worker01 --ignore-daemonsets
```

> ```
> nnode/worker01 cordoned
> WARNING: ignoring DaemonSet-managed Pods: calico-system/calico-node-zdxsp, kube-system/kube-proxy-svv9b, logging/filebeat-filebeat-dgt92, metallb-system/speaker-2nfxc, monitoring/node-exporter-dl47z, projectcontour/envoy-n8bzw
> evicting pod tigera-operator/tigera-operator-657cc89589-tkjc4
> evicting pod calico-system/calico-typha-7c6d988549-9h844
> evicting pod calico-system/calico-kube-controllers-546d44f5b7-wp5bk
> evicting pod kube-system/coredns-74ff55c5b-tbjzv
> evicting pod logging/elasticsearch-master-2
> pod/calico-typha-7c6d988549-9h844 evicted
> pod/calico-kube-controllers-546d44f5b7-wp5bk evicted
> pod/tigera-operator-657cc89589-tkjc4 evicted
> pod/elasticsearch-master-2 evicted
> pod/coredns-74ff55c5b-tbjzv evicted
> node/worker01 evicted
> ```

On another terminal connect with `ssh` inside the worker node:

```bash
ssh worker01
```

Upgrade `kubeadm`, `kubelet` and `kubectl`:

```bash
sudo apt-mark unhold kubeadm kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.20.1-00 kubelet=1.20.1-00 kubectl=1.20.1-00 && \
sudo apt-mark hold kubeadm kubelet kubectl
```

Run `kubeadm` upgrade node:

```bash
sudo kubeadm upgrade node
```

> ```
> [upgrade] Reading configuration from the cluster...
> [upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
> [preflight] Running pre-flight checks
> [preflight] Skipping prepull. Not a control plane node.
> [upgrade] Skipping phase. Not a control plane node.
> [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
> [upgrade] The configuration for this node was successfully updated!
> [upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
> ```

Return on your `student` terminal and `uncordon` the `worke01`

```bash
kubectl uncordon worker01
```

> ```
> node/k8s-worker01 uncordoned
> ```

Check the version of your nodes:

```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES                  AGE   VERSION
> master01   Ready    control-plane,master   20h   v1.20.1
> master02   Ready    control-plane,master   40m   v1.20.1
> master03   Ready    control-plane,master   30m   v1.20.1
> worker01   Ready    <none>                 19h   v1.20.1
> worker02   Ready    <none>                 19h   v1.19.6
> worker03   Ready    <none>                 19h   v1.19.6
> ```

## Repeat last steps to the worker02 and worker03 nodes to upgrade them.

The final result wil be a complete upgraded cluster:

```bash
kubectl get nodes
```

> ```
> master01   Ready    control-plane,master   20h   v1.20.1
> master02   Ready    control-plane,master   40m   v1.20.1
> master03   Ready    control-plane,master   30m   v1.20.1
> worker01   Ready    <none>                 19h   v1.20.1
> worker02   Ready    <none>                 19h   v1.20.1
> worker03   Ready    <none>                 19h   v1.20.1
> ```

All nodes are now upgraded.
