---
layout: post
title: Multi Master installation
comments: false
category: blog
---
# Multi Master installation


Check the number of master nodes

```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES    AGE   VERSION
> master01   Ready    master   19h   v1.19.6
> worker01   Ready    <none>   18h   v1.19.6
> worker02   Ready    <none>   18h   v1.19.6
> worker03   Ready    <none>   18h   v1.19.6
> ```


On `master01` let's retrive the `join` command:

```bash
ssh master01
```

The command to let other masters join the cluster looks like this but, as you can see, there are missing pieces which
we'll generate shortly!

```bash
sudo kubeadm join master:6443 \
    --token <your-generated token> \
    --discovery-token-ca-cert-hash \
        sha256:<your_sha256-discovery-ca-cert-hash> \
    --control-plane --certificate-key \
        <your_control-plane-key>
```

```bash
kubeadm token create
```

> ```
> j5ofa2.tpjfosco1asd66sh
> ```

```bash
openssl x509 -pubkey \
    -in /etc/kubernetes/pki/ca.crt | openssl rsa \
    -pubin -outform der 2>/dev/null | openssl dgst \
    -sha256 -hex | sed 's/^.* //'
```

> ```
> 020c6df64ae4e5ecaa996dc519ce758447235481853eca77179a703c22e9a8a0
> ```

You should add `sha256:` to this string:

> ```
> sha256:020c6df64ae4e5ecaa996dc519ce758447235481853eca77179a703c22e9a8a0
> ```

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

> ```
> W0123 21:38:00.801723   28408 validation.go:28] Cannot validate kube-proxy config - no validator is available
> W0123 21:38:00.801926   28408 validation.go:28] Cannot validate kubelet config - no validator is available
> [upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
> [upload-certs] Using certificate key:
> ae26b89ad27949b8db193372638661617a8c0439468b11d4a0505a0d8bedc3a4
> ```

The final command looks like this, comprehensive of what we generated above:

```bash
sudo kubeadm master:6443 \
    --token j5ofa2.tpjfosco1asd66sh \
    --discovery-token-ca-cert-hash \
        sha256:020c6df64ae4e5ecaa996dc519ce758447235481853eca77179a703c22e9a8a0 \
    --control-plane --certificate-key \
        ae26b89ad27949b8db193372638661617a8c0439468b11d4a0505a0d8bedc3a4
```

### Install kubeadm, kubectl, kubelet on the master02.

Open a new terminal tab and connect to `master02`

```bash
ssh master02
```

Let's start by adding the Kubernetes signing key:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg \
| sudo apt-key add -
```

> ```
> OK
> ```

Add new repo for K8s. You could also get a tar file or use code from GitHub.
Create the file and add an entry for the `main` repo for your distribution.

NOTE: At the time of writing only Ubuntu 20.04 Xenial Kubernetes repository is available.
Replace the below xenial with bionic codename once the Ubuntu 20.04 Kubernetes repository becomes available.
https://kubernetes.io/docs/setup/independent/install-kubeadm/

```bash
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

> ```
> <output_omitted>
> ```


```bash
sudo apt-get install -y kubeadm=1.19.6-00 kubelet=1.19.6-00 kubectl=1.19.6-00
```

> ```
> <output_omitted>
> ```

Now put the package in hold state, so a global update do not change the version of this application:

```bash
sudo apt-mark hold kubeadm kubectl kubelet
```

> ```
> kubeadm set on hold.
> kubectl set on hold.
> kubelet set on hold.
> ```

Now run the command obtained in previus steps on `master02` with this type of command:

```bash
sudo kubeadm join master:6443 \
    --token j5ofa2.tpjfosco1asd66sh \
    --discovery-token-ca-cert-hash \
        sha256:020c6df64ae4e5ecaa996dc519ce758447235481853eca77179a703c22e9a8a0 \
    --control-plane --certificate-key \
        ae26b89ad27949b8db193372638661617a8c0439468b11d4a0505a0d8bedc3a4
```

> ```
> [preflight] Running pre-flight checks
> [preflight] Reading configuration from the cluster...
> [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
> [preflight] Running pre-flight checks before initializing the new control plane instance
> [preflight] Pulling images required for setting up a Kubernetes cluster
> <otput_omitted>
> ```


On the `student` client machine check the nodes:

```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES    AGE    VERSION
> master01   Ready    master   20h    v1.19.6
> master02   Ready    master   103s   v1.19.6
> worker01   Ready    <none>   18h    v1.19.6
> worker02   Ready    <none>   18h    v1.19.6
> worker03   Ready    <none>   18h    v1.19.6
> ```

### Repeat the steps that you performed with master02 on the master03 to join it

Open a new terminal tab and connect to `master03`

```bash
ssh master03
```

**Now install Kubelet, Kubeadm,Kubectl, hold the packages and run the join command**

The final situation will be the following. On the student machine check if the 3 masters are available:

```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES    AGE    VERSION
> master01   Ready    master   20h    v1.19.6
> master02   Ready    master   11m    v1.19.6
> master03   Ready    master   114s   v1.19.6
> worker01   Ready    <none>   18h    v1.19.6
> worker02   Ready    <none>   18h    v1.19.6
> worker03   Ready    <none>   18h    v1.19.6
> ```


Now you have an HA-Cluster.
