---
layout: post
title: Install Kubernetes
comments: false
category: blog
---
# Install Kubernetes


## Overview

There are several Kubernetes installation tools provided by various vendors. In this lab we will learn to use `kubeadm`.
As a community-supported independent tool, it is planned to become the primary manner to build a Kubernetes cluster.

The labs were written using Ubuntu instances running on cloud. They have been written to be vendor-agnostic so could run
on AWS, local hardware, or inside of virtualization system to give you the most flexibility and options. Each platform
will have different access methods and considerations.

If using your own equipment you will have to disable swap on every node. There may be other requirements which will be
shown as warnings or errors when using the `kubeadm` command. While most commands are run as a regular user, there are
some which require root privilege.

In the following exercise we will install Kubernetes on a single node then grow the cluster, adding more compute
resources.

Various exercises will use YAML files, which are included in the text. You are encouraged to write the files when
possible, as the syntax of YAML has white space indentation requirements that are important to learn. An important note,
do not use tabs in your YAML files, white spaces only. Indentation matters.

## Access on VM
You can access using labs webpage:

https://labs.desotech.it

Username: provided by your instructor

Password: provided by your instructor


## Install Kubernetes

Connect to client node and open terminal for SSH to the fist master:
```bash
ssh master01
```


### Install kubeadm, kubectl, kubelet

Let's start by adding the Kubernetes signing key:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg \
| sudo apt-key add -
```

> ```bash
> OK
> ```

Add a new repo for k8s. You could also get a tar file or use the code from GitHub.
Create the file and add an entry for the main repo for your distribution.

NOTE: At the time of writing only the Ubuntu 20.04 Xenial Kubernetes repository is available.
Replace the below `xenial` with `bionic` codename once the Ubuntu 20.04 Kubernetes repository becomes available.
https://kubernetes.io/docs/setup/independent/install-kubeadm/

```bash
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

```bash
sudo apt-get update
```

Install the software. There are regular releases the newest of which can be used by omitting the equal sign and version
information on the command line. Historically new versions have lots of changes and a good chance of containing bugs.

```bash
sudo apt-get install -y kubeadm=1.19.6-00 kubelet=1.19.6-00 kubectl=1.19.6-00
```

Now put the package in hold state, so update do not change the version of this application:

```
sudo apt-mark hold kubeadm kubectl kubelet
```

> ```
> kubeadm set on hold.
> kubectl set on hold.
> kubelet set on hold.
> ```

Please note: the output lists several commands which following commands will complete.

Pull Kubernetes images from Google repository:

```bash
sudo kubeadm config images pull
```

> ```bash
>  [config/images] Pulled k8s.gcr.io/kube-apiserver:v1.19.6
>  [config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.19.6
>  [config/images] Pulled k8s.gcr.io/kube-scheduler:v1.19.6
>  [config/images] Pulled k8s.gcr.io/kube-proxy:v1.19.6
>  [config/images] Pulled k8s.gcr.io/pause:3.2
>  [config/images] Pulled k8s.gcr.io/etcd:3.4.13-0
>  [config/images] Pulled k8s.gcr.io/coredns:1.7.0
> ```

Check the services:

```bash
sudo systemctl status kubelet
```

```
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
```

The service must be active (running).
If the service is not active please enable and start it:

```bash
systemctl enable --now kubelet.service
```

We can deploy our first k8s cluster using `kubeadm`

Fortunately, Kubernetes has a component called the Kubelet which manages containers running on a single host.
It uses the API server but it doesn't depend on it so we can actually use the Kubelet to manage the control plane
components. This is exactly what `kubeadm` sets us up to do. Let's look at what happens when we run `kubeadm`.



## Kubeadm config

The preferred way to configure `kubeadm` is to pass an YAML configuration file with the `--config` option.
Some of the configuration options defined in the kubeadm config file are also available as command line flags, but only
the most common/simple use case are supported with this approach.

A `kubeadm` config file could contain multiple configuration types separated using three dashes (`---`).

kubeadm supports the following configuration types:

> ```yaml
> apiVersion: kubeadm.k8s.io/v1beta2
> kind: InitConfiguration
>
> apiVersion: kubeadm.k8s.io/v1beta2
> kind: ClusterConfiguration
>
> apiVersion: kubelet.config.k8s.io/v1beta1
> kind: KubeletConfiguration
>
> apiVersion: kubeproxy.config.k8s.io/v1alpha1
> kind: KubeProxyConfiguration
>
> apiVersion: kubeadm.k8s.io/v1beta2
> kind: JoinConfiguration
> ```

The list of configuration types that must be included in a configuration file depends on the action you are performing
(`init` or `join`) and on the configuration options you are going to use (defaults or advanced customization).

If some configuration types are not provided, or provided only partially, `kubeadm` will use default values; the
defaults provided by `kubeadm` also include enforcing consistency of values across components when required (e.g.
cluster-cidr flag on controller manager and clusterCIDR on kube-proxy).

Users are always allowed to override the default values, with the only exception of a small subset of settings with
relevance to security (e.g. enforcing authorization-mode Node and RBAC on the api server)

If the user provides a configuration types that is not expected for the action you are performing, `kubeadm` will ignore
those types and print a warning.


## Kubeadm init configuration types

When executing `kubeadm init` with the `--config` option, the following configuration types could be used:
`InitConfiguration`, `ClusterConfiguration`, `KubeProxyConfiguration`, `KubeletConfiguration`, but only one between
`InitConfiguration` and `ClusterConfiguration` is mandatory.

### InitConfiguration

> ```yaml
> apiVersion: kubeadm.k8s.io/v1beta2
> kind: InitConfiguration
> bootstrapTokens:
> #...
> nodeRegistration:
> #...
> ```

The `InitConfiguration` type should be used to configure runtime settings, that in case of `kubeadm init` are the
configuration of the bootstrap token and all the settings which are specific to the node where `kubeadm` is executed,
including:

- nodeRegistration, which holds fields that relate to registering the new node to the cluster; use it to customize the
  node name, the CRI socket to use or any other settings that should apply to this node only (e.g. the node ip).

- LocalAPIEndpoint, which represents the endpoint of the instance of the API server to be deployed on this node; use it
  e.g. to customize the API server advertise address.


### ClusterConfiguration

> ```yaml
> apiVersion: kubeadm.k8s.io/v1beta2
> kind: ClusterConfiguration
> networking:
> #...
> etcd:
> #...
> apiServer:
>   extraArgs:
>   #...
>   extraVolumes:
>   #...
> #...
> ```

The `ClusterConfiguration` type should be used to configure cluster-wide settings, including settings for:

- Networking, that holds configuration for the networking topology of the cluster; use it e.g. to customize node subnet
  or services subnet.

- Etcd configurations; use it e.g. to customize the local etcd or to configure the API server for using an external etcd
  cluster.

- `kube-apiserver`, `kube-scheduler`, `kube-controller-manager` configurations; use it to customize control-plane
  components by adding customized setting or overriding `kubeadm` default settings.

### KubeProxyConfiguration

> ```yaml
> apiVersion: kubeproxy.config.k8s.io/v1alpha1
> kind: KubeProxyConfiguration
>    ...
> ```

The `KubeProxyConfiguration` type should be used to change the configuration passed to `kube-proxy` instances deployed
in the cluster. If this object is not provided or provided only partially, `kubeadm` applies defaults.


### KubeletConfiguration

> ```yaml
> apiVersion: kubelet.config.k8s.io/v1beta1
> kind: KubeletConfiguration
> #...
> ```

The `KubeletConfiguration` type should be used to change the configurations that will be passed to all kubelet instances
deployed in the cluster. If this object is not provided or provided only partially, `kubeadm` applies defaults.

### Create a new ClusterConfiguration

Now we can create a file with the following configuration:

To print the defaults for `init` and `join` actions use the following commands:

```bash
kubeadm config print init-defaults
kubeadm config print join-defaults
```

Here a simple reference:

- `podSubnet` will be the IP address of the pod
- `serviceSubnet` will be the IP assigned to the load balancer of k8s
- `dnsDomain` will be the domain of our cluster
- `kubernetesVersion` will specify the version you want to install
- `controlPlaneEndpoint` will address you to use the control plane's endpoint to `master:6443`. The hostname `master`
  will be the master host, but that should be the external load balancer if you use high availability.
- `clusterName` will be the name of the k8s cluster.

```bash
cd ~
vi /home/student/ClusterConfiguration.yaml
```

File: `/home/student/ClusterConfiguration.yaml`

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "172.16.0.1/23"
  dnsDomain: "cluster.local"
kubernetesVersion: "v1.19.6"
controlPlaneEndpoint: "master:6443"
clusterName: "desotech"
```

```bash
sudo kubeadm init --config /home/student/ClusterConfiguration.yaml --upload-certs | tee /home/student/kubeadm-init.out
```

- **--upload-certs**: Upload control-plane certificates to the kubeadm-certs Secret.

> ```
>  # ...
>  Your Kubernetes control-plane has initialized successfully!
>
>  To start using your cluster, you need to run the following as a regular user:
>
>    mkdir -p $HOME/.kube
>    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
>    sudo chown $(id -u):$(id -g) $HOME/.kube/config
>
>  You should now deploy a pod network to the cluster.
>  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>    https://kubernetes.io/docs/concepts/cluster-administration/addons/
>
>  You can now join any number of the control-plane node running the following command on each as root:
>
>    kubeadm join master:6443 --token em09me.ip7b0nxx265scgpl \
>      --discovery-token-ca-cert-hash sha256:e83303b212d7831a3803ad81dc3375cd8781bfe6de2387c27b5b0109c249f8b3 \
>      --control-plane --certificate-key 1feef3980e1a9c25c2edc0006bd888287efad5777e9aa0fed1347cea308764a9
>
>  Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
>  As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
>  "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
>
>  Then you can join any number of worker nodes by running the following on each as root:
>
>  kubeadm join master:6443 --token em09me.ip7b0nxx265scgpl \
>      --discovery-token-ca-cert-hash sha256:e83303b212d7831a3803ad81dc3375cd8781bfe6de2387c27b5b0109c249f8b3
> ```
