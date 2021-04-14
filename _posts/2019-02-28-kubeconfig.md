---
layout: post
title: kubeconfig
comments: false
category: blog
---
# kubeconfig


As suggested in the directions at the end of the previous output we will allow a non-root user admin level access to the
cluster. Take a quick look at the configuration file once it has been copied and the permissions fixed.

As a regular user, try to list k8s nodes

```bash
kubectl get nodes
```
> ```
> The connection to the server localhost:8080 was refused - did you specify the right host or port?
> ```
As you can see, you cannot connect to the server because you do not have kubeconfig set.
We copy the kubeconfig file from `/etc/kubernetes/admin.conf` to `/home/student/.kube/config`

as a regular user
> ```
> mkdir -p $HOME/.kube
> ```

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubectl config view
```

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://master:6443
  name: desotech
contexts:
- context:
    cluster: desotech
    user: kubernetes-admin
  name: kubernetes-admin@desotech
current-context: kubernetes-admin@desotech
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

You can find the same output, without the certificate being redacted in the `~/.kube/config` file.

```bash
less .kube/config
```

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <CERTIFICATE>
    server: https://master:6443
  name: desotech
contexts:
- context:
    cluster: desotech
    user: kubernetes-admin
  name: kubernetes-admin@desotech
current-context: kubernetes-admin@desotech
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <CERTIFICATE>
    client-key-data: <CERTIFICATE>
```

Running the same `kubectl` command now should allow you to view the nodes.

```bash
kubectl get nodes
```

> ```
> NAME       STATUS     ROLES    AGE     VERSION
> master01   NotReady   master   6m25s   v1.19.6
> ```

Now go to the `student` machine:

```bash
ssh student@student
```

```bash
mkdir -p $HOME/.kube
```

```bash
scp student@master01:.kube/config .kube/config
```

> ```
> config      100% 5550   416.2KB/s   00:00
> ```

View the current context for `kubectl`

```bash
kubectl config current-context
```

```bash
kubernetes-admin@desotech
```

For display clusters defined in the kubeconfig

```bash
kubectl config get-contexts
```

> ```
> CURRENT   NAME                        CLUSTER    AUTHINFO           NAMESPACE
> *         kubernetes-admin@desotech   desotech   kubernetes-admin
> ```

To view the current cluster, run:

```bash
kubectl config get-clusters
```
> ```
> NAME
> desotech
> ```

## Kubectl TAB completion bash

While many objects have short names, a `kubectl` command can be a lot to type. We will enable `bash` auto-completion.
Start by adding the settings to the current shell. Then update the `~/.bashrc` file to make it persistent.

```bash
source <(kubectl completion bash)
```
```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

Test by describing the node again. Type the first three letters of the sub-command then type the <kbd>TAB</kbd> key.
Auto-completion assumes the default namespace. Pass the namespace first to use auto-completion with a different
namespace. By pressing <kbd>TAB</kbd> multiple times you will see a list of possible values. Continue typing until a
unique name is used. First look at the current node, then look at the pods in the `kube-system` namespace.

```bash
kubectl des<Tab> na<Tab> de<Tab>
```

> ```
> kubectl describe namespaces default
> ```


```bash
kubectl -n kube-s<Tab> g<Tab> p<Tab>
```

> ```
> kubectl -n kube-system get pod
> ```
