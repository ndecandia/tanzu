---
layout: post
title: ConfigMap
comments: false
category: blog
---
# ConfigMap


By the end of this lab, you will be able to

- Create and attach ConfigMaps for pods

### Working with ConfigMaps (CM)

A `ConfigMap` is a cluster API object used to store non-confidential data in key-value pairs.
A config map can source application configuration data in variety through a variety of methods.
In this lab section, we will examine how the `ConfigMap` object can present such data to a containerized application
running in a pod.

### ConfigMap Data in a Volume

In the following steps, we will configure NGINX to run as reverse-proxy for our custom web application, which is set to
run on TCP port `5000`. In order to do so, we will need to supply the NGINX container with a custom configuration.
In our case, we will provide the configuration through a custom configuration file to be mounted into the NGINX
container.

#### Step 1: Make and move to the `~/configMap/` directory:

```bash
mkdir ~/configMap ; cd ~/configMap
```

#### Step 2: Define the following `ConfigMap` in the file `~/configMap/cm-nginx-conf.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-nginx-conf
data:
  nginx.conf: |-
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        sendfile        on;
        keepalive_timeout  65;

        upstream webapp {
            server 127.0.0.1:5000;
        }

        server {
            listen 80;

            location / {
                proxy_pass         http://webapp;
                proxy_redirect     off;
            }
        }
    }
```

> In this YAML declartion, we specify both the file name and the
> configuration data of the file to be stored within the `ConfigMap`
> object.

#### Step 3: Create the `cm-nginx-conf` Config Map:

```bash
kubectl create -f cm-nginx-conf.yaml
```

> ```
> configmap/cm-nginx-conf created
> ```

#### Step 4: Confirm that the `ConfigMap` was created successfully:

```bash
kubectl describe configmap cm-nginx-conf
```

> ```
> Name:         cm-nginx-conf
> Namespace:    default
> Labels:       <none>
> Annotations:  <none>
>
> Data
> ====
> nginx.conf:
> ----
> user  nginx;
> worker_processes  1;
> ....
> ```

As we can see from the output, the entire contents of the `nginx.conf` is stored in our config map. Our next step is
present this file to our nginx container.

#### Step 5: Define the following multi-container pod in the file: `~/configMap/mc2.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mc2
  labels:
    app: mc2
spec:
  containers:
    - name: webapp
      image: training/webapp
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-proxy-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
  volumes:
    - name: nginx-proxy-config
      configMap:
        name: cm-nginx-conf
```

<!-- The YAML here should look familiar to our `mc1` pod manifest, with clear differences. Notably, in our volumes field, we -->
<!-- declare the `ConfigMap` as the source for the volume to be mounted in the NGINX container. In effect, we should expect -->
<!-- the default configuration file to be replaced with our custom file at time of container creation. -->

> **Note**: Although port `80` for the NGINX container is explicitly defined
> to be exposed to traffic outside the pod, port 5000 is accessible
> outside of the pod as well. By default, any port which is listening on
> the default `0.0.0.0` IP address inside a container will be accessible
> from the pod network.

#### Step 6: Create the `mc2` pod:

```bash
kubectl create -f mc2.yaml
```

> ```
> pod/mc2 created
> ```

#### Step 7: Check on the status of the pod, specifically ensuring that the config map is mounted correctly:

```bash
kubectl describe pod mc2
```

> ```
> Name:         mc2
> Namespace:    default
> Priority:     0
> Node:         node2/10.10.8.196
> ....
> Containers:
> ....
>     nginx:
>     ....
>     Mounts:@g@
>       /etc/nginx/nginx.conf from nginx-proxy-config (rw,path="nginx.conf")@/g@
> ....
> Volumes:@g@
>   nginx-proxy-config:
>   Type: ConfigMap (a volume populated by a ConfigMap)
>   Name: mc2-nginx-conf@/g@
>   .....
> ```

From our abbreviated output, we can confirm that our `ConfigMap` object is mounted as a volume in the nginx container.
As such, we can expect our nginx container to redirect incoming traffic on port 80 to the localhost on port `5000`.

#### Step 8: Get IP of mc2 pod using this command:

```bash
kubectl get pod mc2 -o jsonpath='{.status.podIP}{"\n"}'
```

Take note of the pod's IP and try to `curl` using a temporary pod:

```bash
kubectl run curl --image=radial/busyboxplus:curl --rm -it
```

Now you're inside a temporary pod, try to `curl` the `mc2` pod using its IP:

```bash
curl 10.244.107.183 # Use the IP of your pod you got earlier
```

> ```
> Hello world!
> ```

Well done! If you see `Hello world`, it means that your app works correctly and your `configMap` is good.

#### Step 9: Delete the `mc2` pod:

```bash
kubectl delete pod mc2
```

> ```
> pod "mc2" deleted
> ```

### ConfigMap Data via Environment Variables

In this next lab section, we will configure a `redis` pod using a `ConfigMap` object that injects configuration data as
environment variables for the `redis` application along with supplying command line arguments to the pod startup
script. In the following exercises, we will examine implementing two types of `ConfigMap` object data to a pod to
illustrate how containerized application can consume data from a `ConfigMap` object.

#### Step 1: Define the following `ConfigMap` in the file `~/configMap/redis-dev-cm.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-dev-config
data:
  empty_password: "yes"
  maxmemory: 2mb
  maxmemory-policy: allkeys-lru
```

In this `ConfigMap` we are setting our values for the redis instance to not require a password for making changes along
with configuring redis to run 2 MB of memory, effectively making it run as a caching server. Let’s configure our pod to
use this `ConfigMap`

#### Step 2: Create the `redis-dev-config` config map:

```bash
kubectl create -f redis-dev-cm.yaml
```

> ```
> configmap/redis-dev-config created
> ```

With our `ConfigMap` created, we can now work on our `redis` pod.

#### Step 3: Define the following `redis` pod in the file `~/configMap/redis-dev-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-dev
  labels:
    env: dev
spec:
  containers:
  - name: redis
    image: bitnami/redis:6.0
    command:
    - /opt/bitnami/scripts/redis/run.sh
    - --maxmemory $(MAX_MEMORY)
    - --maxmemory-policy $(MAX_MEMORY_POLICY)
    env:
      - name: ALLOW_EMPTY_PASSWORDS
        valueFrom:
          configMapKeyRef:
            name: redis-dev-config
            key: empty_password
      - name: MAX_MEMORY
        valueFrom:
          configMapKeyRef:
            name: redis-dev-config
            key: maxmemory
      - name: MAX_MEMORY_POLICY
        valueFrom:
          configMapKeyRef:
            name: redis-dev-config
            key: maxmemory-policy
```

Take a moment to review YAML above. In it, we are providing environment variables that are provided to the application
in two different ways. The intialization scripts of the redis container refers to the `ALLOW_EMPTY_PASSWORDS` variable
to configure the redis instance with no admin password, which is suitable for testing.  However, the `MAX_MEMORY` and
`MAX_MEMORY_POLICY` variables will need to be supplied to shell script as specfied by the documentation. In this case,
the specfic syntax within the `command:` field is how we pass our `ConfigMap` data to our application via the command
line.

#### Step 4: Create the `redis-dev` pod:

```bash
kubectl create -f redis-dev-pod.yaml
```

> ```
> pod/redis-dev created
> ```

#### Step 5: With the pod created, confirm that the `ConfigMap` data was injected into the redis instance correctly:

```bash
kubectl exec -it redis-dev -- redis-cli
```

> ```
> 127.0.0.1:6379> CONFIG GET requirepass
>
> 1) "requirepass"
> 2) ""
>
> 127.0.0.1:6379> CONFIG GET maxmemory
>
> 1) "maxmemory"
> 2) "2097152"
>
> 127.0.0.1:6379> CONFIG GET maxmemory-policy
>
> 1) "maxmemory-policy"
> 2) "allkeys-lru"
> ```

From our output, we can see that configuration from our `ConfigMap` has configured our redis instance appropriately.
As this type of configuration is suitable for a testing or development environment, a follow-up question might be, what
about production? The response to the question about production is specific to the environment and requirements for
production; however, the key point to emphasize is that the application running in the pod **would not** have to be
changed. Rather just the configuration parameters would need to be updated. To illustrate this point, we will adjust
the configuration parameters we provided to the redis application, specifically:

- supplying an administrator password
- adjusting the Max Memory size

The way we will provide this configuration data will require changes to our current Kubernetes manifests.

#### Step 6: Make copies of the existing YAML files replacing `dev` with `prod`:

```bash
cp redis-dev-cm.yaml redis-prod-cm.yaml
```

```bash
cp redis-dev-pod.yaml redis-prod-pod.yaml
```

#### Step 7: Modify the `redis-prod-cm.yaml` file with the following changes:

> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: redis-prod-config
> data:
>   redis_password: supersecret
>   maxmemory: 100mb
>   maxmemory-policy: allkeys-lru
> ```

Step 8: Modify the `redis-prod-pod.yaml` file with the following changes:

> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: redis-prod
>   labels:
>     env: prod
> spec:
>   containers:
>   - name: redis
>     image: bitnami/redis:6.0
>     command:
>     - /opt/bitnami/scripts/redis/run.sh
>     - --requirepass $(REDIS_PASSWORD)
>     - --maxmemory $(MAX_MEMORY)
>     - --maxmemory-policy $(MAX_MEMORY_POLICY)
>     env:
>       - name: REDIS_PASSWORD
>         valueFrom:
>           configMapKeyRef:
>             name: redis-prod-config
>             key: redis_password
>       - name: MAX_MEMORY
>         valueFrom:
>           configMapKeyRef:
>             name: redis-prod-config
>             key: maxmemory
>       - name: MAX_MEMORY_POLICY
>         valueFrom:
>           configMapKeyRef:
>             name: redis-prod-config
>             key: maxmemory-policy
> ```

As we are supplying a password, we need to amend the startup parameters to configure the password value.
The rest of the changes are in line with specifying the `prod` value.

#### Step 9: With the YAML file changes complete, create the `redis` cluster objects:

```bash
kubectl create -f redis-prod-cm.yaml -f redis-prod-pod.yaml
```

> ```
> configmap/redis-prod-config created
> pod/redis-prod created
> ```

With our cluster objects created, we are ready to test our changes.

#### Step 10: Verify `prod` configuration changes in the `redis-prod` instance:

```bash
kubectl exec -it redis-prod -- redis-cli
```

> ```
> 127.0.0.1:6379> auth supersecret
>
> OK
>
> 127.0.0.1:6379> CONFIG GET requirepass
>
> 1) "requirepass"
> 2) "supersecret"
>
> 127.0.0.1:6379> CONFIG GET maxmemory
>
> 1) "maxmemory"
> 2) "104857600"
>
> 127.0.0.1:6379> CONFIG GET maxmemory-policy
>
> 1) "maxmemory-policy"
> 2) "allkeys-lru"
> ```

From the output, we can see that our configuration changes are in place for the `redis-prod` instance.

Delete your two pods:

```bash
kubectl delete pod redis-dev redis-prod
```
