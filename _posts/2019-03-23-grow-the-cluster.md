---
layout: post
title: Grow the Cluster
comments: false
category: blog
---
# Grow the Cluster


Open another terminal and connect into a your worker node. Install Docker and Kubernetes software.
These are the many, but not all, of the steps we did on the master node.

The book will use the worker prompt for the node being added to help keep track of the proper node for each command.
Note that the prompt indicates both the user and system upon which run the command.

## Connect to worker node:

Become root and update and upgrade the system. Answer any questions to use the defaults.

```bash
ssh student@worker01
```

## Check if Docker is active:

```bash
systemctl is-active docker
```

> ```
> active
> ```

## Install kubeadm, kubectl, kubelet

Let's start by adding the Kubernetes signing key:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg \
| sudo apt-key add -
```

> ```bash
> OK
> ```

Add the new repo for k8s. You could also get a tar file or use the code from GitHub. Create the file and add an entry
for the `main` repo for your distribution.

```bash
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

> ```bash
> <output_omitted>
> ```

Install `kubeadm` and `kubelet`

```bash
sudo apt-get install -y kubeadm=1.19.6-00 kubelet=1.19.6-00 kubectl=1.19.6-00
```

> ```bash
> <output_omitted>
> ```

Mark as hold the packages installed

```bash
sudo apt-mark hold kubeadm kubectl kubelet
```

> ```
> kubeadm set on hold.
> kubectl set on hold.
> kubelet set on hold.
> ```

Please note: the output lists several commands which the following commands will complete.

Now open another terminal but leave this terminal session open.

On the new console, access to the master node:

```bash
ssh student@master01
```

and run the following command for list the tokens.

```bash
sudo kubeadm token list
```

```bash
sudo kubeadm token list
```
> ```
> TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
> b7z649.hdab6ufm9orgiod9   1h          2020-12-19T17:49:13Z   <none>                   Proxy for managing TTL for the kubeadm-certs secret        <none>
> em09me.ip7b0nxx265scgpl   23h         2020-12-20T15:49:13Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
> ```

Only if the token has expired, you can create a new token, to use as part of the `join` command.

```bash
sudo kubeadm token create
```

> ```
> u3a427.b65xcvlbn8s6dja6
> ```

Come back the worker node's console and open a new file with `vi`:

```bash
vi join-command
```

Insert a new line with `o`
and note the token obtained from previous command:

```
token: u3a427.b65xcvlbn8s6dja6
```

Type `:w` and do not close this terminal.


Starting in v1.9 you should create and use a Discovery Token CA Cert Hash created from the master to ensure the node
joins the cluster in a secure manner. Run this on the master node or wherever you have a copy of the CA file. You will
get a long string as output.

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex | sed 's/^.* //'
```

> ```
> ed4a67aca3ef58114483b97d112ea223e1335e1a5f7197a22c9f41e6b4cc53fd
> ```


Insert a new line with `o` under the token line:


```yaml
discovery-token-ca-cert-hash: ed4a67aca3ef58114483b97d112ea223e1335e1a5f7197a22c9f41e6b4cc53fd
```

Now save the file with `:wq` and exit:

```bash
cat join-command
```

You should see an output like this:

> ```
> token: u3a427.b65xcvlbn8s6dja6
> discovery-token-ca-cert-hash: ed4a67aca3ef58114483b97d112ea223e1335e1a5f7197a22c9f41e6b4cc53fd
> ```

A worker can join a Kubernetes control plane with the `kubeadm join` command, which requires the following parameters:

- The token
- Master node IP or FQDN and the port where the API server is exposed
- The SHA256 hash of the discovery token CA cert

On your worker terminal type:

```bash
sudo kubeadm join --token <token> master:6443 \
	--discovery-token-ca-cert-hash sha256:<discovery-token-ca-cert-hash>
```

Once written, start this command, and the worker node will join the kubernetes cluster:

```bash
sudo kubeadm join --token u3a427.b65xcvlbn8s6dja6 master:6443 \
	--discovery-token-ca-cert-hash sha256:ed4a67aca3ef58114483b97d112ea223e1335e1a5f7197a22c9f41e6b4cc53fd \
	| tee kubeadm-join.out
```

This command takes a few minutes to run. It'll output something similar to this:

> ```
> This node has joined the cluster:
> * Certificate signing request was sent to apiserver and a response was received.
> * The Kubelet was informed of the new secure connection details.
>
> Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
> ```


Now open a new terminal from `student` desktop and run the following command:

```bash
kubectl get pod -w
```

> ```
> kubectl get nodes -w
> NAME       STATUS   ROLES    AGE    VERSION
> master01   Ready    master   80m    v1.19.6
> worker01   Ready    <none>   4m4s   v1.19.6
> ```


You can obtain the same result using a special command that generate a new token and release you a `join` command:

```bash
sudo kubeadm token create --print-join-command 2> /dev/null
```

> ```
> kubeadm join master:6443 --token 8kbgno.nrw58r6x9qy9680z \
	--discovery-token-ca-cert-hash sha256:e83303b212d7831a3803ad81dc3375cd8781bfe6de2387c27b5b0109c249f8b3
> ```

You can use this command to join node `worker02` and `worker03` on the kubernetes cluster.

Repeat the steps executing on `worker02` and `worker03`.

When done you should see on your `student` desktop this output:

```bash
kubectl get nodes
```

> ```
> NAME       STATUS   ROLES    AGE    VERSION
> master01   Ready    master   91m    v1.19.6
> worker01   Ready    <none>   14m    v1.19.6
> worker02   Ready    <none>   2m9s   v1.19.6
> worker03   Ready    <none>   44s    v1.19.6
> ```
