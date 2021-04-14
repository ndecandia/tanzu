---
layout: post
title: Introduction Monitoring
comments: false
category: blog
---
# Introduction Monitoring


Monitoring is a crucial aspect of any Ops pipeline and for technologies like Kubernetes which is a rage right now, a
robust monitoring setup can bolster your confidence to migrate production workloads from VMs to Containers.

Today we will deploy a Production grade Prometheus based monitoring system, in less than 5 minutes.

## Prerequisites

1. Running a Kubernetes cluster with at least 6 cores and 8 GB of available memory.
   I will be using a 2 node (1 master and 1 worker) for this lab.
2. Working knowledge of Kubernetes Deployments and Services.

## Setup

1. `Prometheus` server with persistent volume. This will be our metric storage (TSDB).
2. `Alertmanager` server which will trigger alerts to Slack/Hipchat and/or Pagerduty/Victorops etc.
3. `Kube-state-metrics` server to expose container and pod metrics other than those exposed by `cadvisor` on the nodes.
4. `Node-Exporter` server to expose instances metrics - CPU, memory, network.
5. `Grafana` server to create dashboards based on prometheus data.


![monitoring](images/monitoring.jpeg)

```bash
kubectl get nodes
```

> ```
> NAME           STATUS   ROLES    AGE    VERSION
> k8s-master01   Ready    master   238d   v1.19.2
> k8s-master02   Ready    master   238d   v1.19.2
> k8s-master03   Ready    master   238d   v1.19.2
> k8s-worker01   Ready    <none>   94d    v1.19.2
> k8s-worker02   Ready    <none>   94d    v1.19.2
> k8s-worker03   Ready    <none>   94d    v1.19.2
> ```

Clone a repository from GitHub. This repository will contain all the files we will use during our courses:

```bash
cd ˜
git clone https://github.com/desotech-it/DSK201-public.git
```

> ```
> Cloning into 'DSK201-public'...
> ```

```bash
cd /home/student/DSK201-public/monitoring
```

Apply the manifests:

```bash
kubectl apply -f 00-prerequisite/namespace.yaml
```

> ```
> namespace/monitoring created
> ```

Change the context to the `monitoring` namespace:

```bash
kubectl config set-context --current --namespace monitoring
```


## Deploy AlertManager

### What is it AlertManager

Monitoring is incomplete without alerting. We have already looked in monitoring with Prometheus in the previous
article.

`AlertManager` is a single binary which handles alerts sent by Prometheus server and notifies end user.

The `Alertmanager` handles alerts sent by client applications such as the Prometheus server. It takes care of
deduplicating, grouping, and routing them to the correct receiver integration such as email, `PagerDuty`, or `OpsGenie`.
It also takes care of silencing and inhibition of alerts.

The following describes the core concepts the Alertmanager implements.
Consult the configuration documentation to learn how to use them in more detail.

### Deploying Alertmanager

```bash
kubectl apply -f 01-alertmanager/
```

> ```
> configmap/alertmanager created
> deployment.apps/alertmanager created
> service/alertmanager created
> ```

This will create the following:

1. Config-map to be used by alertmanager to manage channels for alerting.
2. Alertmanager deployment with 1 replica running.
3. Service with Nodeport/Loadbalancer IP.

List all pods with label `app=alertmanager` inside namespace `monitoring`

```bash
kubectl get pods -l app=alertmanager -n monitoring
```

```console
NAME                            READY   STATUS    RESTARTS   AGE
alertmanager-5d87b89668-mt62g   1/1     Running   0          24s
```

You should see a pod running and ready 1/1.

List all kuberentes services with label `app=alertmanager` inside namespace `monitoring`

```bash
kubectl get svc -l name=alertmanager -n monitoring
```

```console
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)          AGE
alertmanager   LoadBalancer   10.111.161.115   10.10.99.120   9093:30922/TCP   50s
```

You should see a `LoadBalancer` type.

```bash
kubectl get configmap -n monitoring
```

> ```
> NAME           DATA   AGE
> alertmanager   1      7m8s
> ```

In your browser, navigate to your assigned External IP and you should see the alertmanager console. (10.10.95.205:9093)


We will not use `Alertmanager` in this course. You just learn to deploy it.


## Deploy Kube State Metrics

`kube-state-metrics` is a simple service that listens to the Kubernetes API server and generates metrics about the
state of the objects. It is not focused on the health of the individual Kubernetes components, but rather on the health
of the various objects inside, such as deployments, nodes and pods.

The metrics are exported through the Prometheus golang client on the `/metrics` HTTP endpoint on the listening port
(default 80). They are served either as plaintext or protobuf depending on the `Accept` header. They are designed to be
consumed either by Prometheus itself or by a scraper that is compatible with scraping a Prometheus client endpoint. You
can also open `/metrics` in a browser to see the raw metrics.

```bash
kubectl apply -f 02-kube-state-metrics/
```

> ```
> clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
> serviceaccount/kube-state-metrics created
> clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
> deployment.apps/kube-state-metrics created
> service/kube-state-metrics created
> ```

This will create the following:

1. Service account, cluster-role and cluster-role-binding needed for kube-state-metrics.
2. Kube-state-metrics deployment with 1 replica running.
3. In-cluster service which will be scraped by prometheus for metrics. ( Note the annotation attached to it. )

```bash
kubectl get pods -l app.kubernetes.io/name=kube-state-metrics -n monitoring
```

> ```
> NAME                                  READY   STATUS    RESTARTS   AGE
> kube-state-metrics-6d8cf88456-k9fhc   0/1     Running   0          20s
> ```

Check the services:

```bash
kubectl get svc -l app.kubernetes.io/name=kube-state-metrics -n monitoring
```

> ```
> NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
> kube-state-metrics   ClusterIP   None         <none>        8080/TCP,8081/TCP   14m
> ```

This is a `ClusterIP` service, it will be used by Prometheus for use `kube-state-metrics`.


## Deploy Prometheus

### What is Prometheus?

Prometheus is an open-source system monitoring and alerting toolkit originally built at SoundCloud.
Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very
active developer and user community. It is now a standalone open source project and maintained independently of any
company. To emphasize this, and to clarify the project's governance structure, Prometheus joined the Cloud Native
Computing Foundation in 2016 as the second project hosted, after Kubernetes.

### Features
Prometheus's main features are:

- a multi-dimensional data model with time series data identified by metric name and key/value pairs
- PromQL, a flexible query language to leverage this dimensionality
- no reliance on distributed storage; single server nodes are autonomous
- time series collection happens via a pull model over HTTP
- pushing time series is supported via an intermediary gateway
- targets are discovered via service discovery or static configuration
- multiple modes of graphing and dashboarding support

### Components
The Prometheus ecosystem consists of multiple components, many of which are optional:

- the main Prometheus server which scrapes and stores time-series data
- client libraries for instrumenting application code
- a push gateway for supporting short-lived jobs
- special-purpose exporters for services like HAProxy, StatsD, Graphite, etc.
- an alertmanager to handle alerts
- various support tools

Most Prometheus components are written in Go, making them easy to build and deploy as static binaries.

### Architecture

This diagram illustrates the architecture of Prometheus and some of its ecosystem components:

Prometheus scrapes metrics from instrumented jobs, either directly or via an intermediary push gateway for short-lived
jobs. It stores all scraped samples locally and runs rules over this data to either aggregate and record new time
series from existing data or generate alerts. Grafana or other API consumers can be used to visualize the collected
data.

### When does it fit?

Prometheus works well for recording any purely numeric time series. It fits both machine-centric monitoring as well as
monitoring of highly dynamic service-oriented architectures. In a world of microservices, its support for
multi-dimensional data collection and querying is a particular strength.

Prometheus is designed for reliability, to be the system you go to during an outage to allow you to quickly diagnose
problems. Each Prometheus server is standalone, not depending on network storage or other remote services. You can rely
on it when other parts of your infrastructure are broken, and you do not need to setup extensive infrastructure to use
it.

### When does it not fit?

Prometheus values reliability. You can always view what statistics are available about your system, even under failure
conditions. If you need 100% accuracy, such as for per-request billing, Prometheus is not a good choice as the
collected data will likely not be detailed and complete enough. In such a case you would be best off using some other
system to collect and analyze the data for billing, and Prometheus for the rest of your monitoring.

## Deploying Prometheus

Before deploying, please create a PV and a PVC and name it as `prometheus-volume` (This is important because the pvc
will look for a volume in this name).

Deploy a dynamic persistent volume claim to save Prometheus data:

File: `/home/student/DSK201-public/monitoring/00-prerequisite/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storageclass
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f /home/student/DSK201-public/monitoring/00-prerequisite/pvc.yaml
```

> ```
> persistentvolumeclaim/prometheus-claim created
> ```

```bash
kubectl get pvc
```

> ```
> NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
> prometheus-claim   Bound    pvc-2b4f4d56-2670-43bc-8ccd-fc7b2396c2e5   10Gi       RWO            nfs-storageclass   2s
> ```

```bash
kubectl get pv
```

> ```
> NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                 STORAGECLASS       REASON   AGE
> pvc-2b4f4d56-2670-43bc-8ccd-fc7b2396c2e5   10Gi       RWO            Delete           Bound    monitoring/prometheus-claim                           nfs-storageclass            43s
> ```


```bash
kubectl apply -f 03-prometheus
```

> ```
> serviceaccount/monitoring created
> clusterrolebinding.rbac.authorization.k8s.io/monitoring created
> configmap/prometheus-server-conf created
> configmap/prometheus-rules created
> persistentvolumeclaim/prometheus-claim created
> deployment.apps/prometheus-deployment created
> service/prometheus-service created
> ```

This will create the following:

1. Service account, cluster-role and cluster-role-binding needed for prometheus.
2. Prometheus config map which details the scrape configs and alertmanager endpoint. It should be noted that we can
   directly use the alertmanager service name
3. Prometheus config map for the alerting rules. Some basic alerts are already configured in it (Such as High CPU and
   Mem usage for Containers and Nodes etc). Feel free to add more rules according to your use case.
4. Storage class, persistent volume and persistent volume claim for the prometheus server data directory. This ensures
   data persistence in case the pod restarts.
5. Prometheus deployment with 1 replica running.
6. Service with Loadbalancer IP which can be accessed directly.

```bash
kubectl get pods -l app=prometheus-server -n monitoring
```

> ```
> NAME                                     READY   STATUS    RESTARTS   AGE
> prometheus-deployment-86749954b8-56j2k   1/1     Running   0          2m54s
> ```

```bash
kubectl get svc -l name=prometheus -n monitoring
```

> ```
> NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
> prometheus-service   LoadBalancer   10.97.132.86   10.10.99.121   8080:32605/TCP   50m
> ```

```bash
kubectl get configmap -n monitoring
```

> ```
> NAME                     DATA   AGE
> alertmanager             1      28m
> prometheus-rules         1      10m
> prometheus-server-conf   1      10m
> ```

In your browser, navigate to yor assigned External_IP `http://10.10.95.206:8080` and you should see the prometheus
console.

Please write down this link referenced as Prometheus Console, we will use it later.


It should be noted that under the `Status > Targets` section all the scraped endpoints are visible and under `Alerts`
section all the configured alerts can be seen.

![monitoring](images/prometheus01.png)


## Deploy Node Exporter

Have you ever wondered how you can monitor your entire Linux system performance easily?

What if you wanted to monitor your filesystems, disks, CPUs, but also your network statistics, similarly to what you
would do with `netstat`?

The Prometheus Node Exporter exposes a wide variety of hardware and kernel-related metrics.

![Node-Exporter](images/node-exporter01.png)

Prometheus scrapes targets and the node exporter is just one of them.

Kubernetes keeps track of many metrics that Prometheus can read by default.
Even so, there's often more information we want to know about our cluster and infrastructure that needs to be
translated to a compatible format for Prometheus. To do so, Prometheus uses exporters, small programs that read metrics
from other sources and translate them to the Prometheus format. The node exporter can read system-level statistics
about bare-metal nodes or virtual machines or Containers and export them for Prometheus.

A Node Exporter runs as a `systemd` service that will periodically (every second) gather all the metrics of your system.

We will install our Node-Exporter inside Kubernetes and we will expose device as `dev`, `proc`, `sys` and `rootfs` as
mounted filesystems inside our pod that will run a `node-exporter`.

Enter in your Prometheus Expression Browser checking again your svc


```bash
kubectl get svc -l name=prometheus -n monitoring
```

> ```
> NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
> prometheus-service   LoadBalancer   10.97.132.86   10.10.99.121   8080:32605/TCP   50m
> ```

In my example are `http://10.10.99.121:8080/`

Click on `Status > Target`:

![Prometheus-Target](images/prometheus03.png)

You should have 3 targets:

- `kubernetes-apiservers`
- `kubernetes-nodes-cadvisor`
- `kubernetes-service-endpoints`

These targets have been added by Prometheus configuration file.
You can find the configuration describing your `prometheus-server-conf` configmap:

```bash
kubectl get configmap -n monitoring
```

> ```
> NAME                     DATA   AGE
> alertmanager             1      28m
> prometheus-rules         1      10m
> prometheus-server-conf   1      10m
> ```

```bash
kubectl describe configmap -n monitoring prometheus-server-conf
```

You can see the parameter `job_name` with the target shown above.

You also can see other `job_name` that are not on the list.
This is due to the fact, that there aren't any exporters that respect scrape configs requirements.

Using a `DaemonSet`, `Kubernetes` can run one node exporter per cluster node, and expose the node exporter as a
service.

```bash
kubectl apply -f 04-node-exporter/
```

> ```
> daemonset.apps/node-exporter created
> ```

Check your node exporter pod created.

```bash
kubectl get pod -l k8s-app=node-exporter
```

> ```
> NAME                  READY   STATUS    RESTARTS   AGE
> node-exporter-74zrm   1/1     Running   0          21s
> node-exporter-jqmct   1/1     Running   0          21s
> node-exporter-k4c7v   1/1     Running   0          21s
> node-exporter-mzpf4   1/1     Running   0          21s
> node-exporter-rh4vp   1/1     Running   0          21s
> node-exporter-ztfmw   1/1     Running   0          21s
> ```

We have 6 pods because of 3 control plane node and 3 workers.
Everytime we add a new node, `DaemonSet` will add a new pod.

Open again your Prometheus Expression Browser

`http://10.10.95.206:8080/`

Click on `Status > Target`

![Prometheus-Target](images/prometheus04.png)

That's it – nothing else is necessary to get your machine-level metrics into Prometheus.
With the configuration we used in our example, Prometheus will automatically scrape all node exporters for metrics,
once they are deployed. You can verify this by navigating to the targets page in the Prometheus Expression Browser.

## Deploy Grafana

Grafana is an opensource solution for running data analytics, pulling up metrics that make sense of the massive amount
of data and to monitor our apps with the help of cool customizable dashboards.

Grafana connects with every possible data source, commonly referred to as databases such as Graphite, Prometheus,
Influx DB, ElasticSearch, MySQL, PostgreSQL etc.

Grafana being an open source solution also enables us to write plugins from scratch for integration with several
different data sources.

The tool helps us study, analyse & monitor data over a period of time, technically called timeseries analytics.

It helps us track the user behaviour, application behaviour, frequency of errors popping up in production or a pre-prod
environment, type of errors popping up & the contextual scenarios by providing relative data.

A big upside of the project is that it can be deployed on-prem by organizations which do not want their data to be
streamed over to a vendor cloud, for whatever reason.

Over time this framework has gained a lot of popularity in the industry and is deployed by companies such as PayPal,
eBay, Intel & many more.

Besides the core open-source solution there are other two services offered by the Grafana team for businesses known as
the Grafana Cloud & the Enterprise.

### Deploying Grafana

By now, we have deployed the core of our monitoring system (metric scrape and storage), it is time too put it all
together and create dashboards.

```bash
kubectl apply -f 05-grafana/
```

> ```
> configmap/grafana-datasources created
> persistentvolumeclaim/grafana-claim created
> deployment.apps/grafana created
> service/grafana created
> ```

This will create the following:

1. Grafana deployment with 1 replica running.
2. Service with Loadbalancer IP, which can be accessed directly.

```bash
kubectl get pods -n monitoring -l k8s-app=grafana
```

> ```
> NAME                       READY   STATUS    RESTARTS   AGE
> grafana-5b69c76dc7-rhkpk   1/1     Running   0          23s
> ```

Note: We are using the Prometheus service name in the URL section because both Grafana and Prometheus servers are
deployed in the same cluster. In case the grafana server is outside the cluster, then you should use the prometheus
service’s external IP in the URL.

```bash
kubectl get svc -n monitoring -l k8s-app=grafana
```

> ```
> NAME      TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)          AGE
> grafana   LoadBalancer   10.99.59.2      10.10.95.207      3000:31426/TCP   69s
> ```

In my example I can access using this URL: http://10.10.95.207:3000


Use the following default username and password to login. Once you login with default credentials, it will prompt to
change the default password.

`User: admin`
`Pass: admin`

Re-enter the same username and password:

`User: admin`
`Pass: admin`


You have completed the installation.

Click Configuration link:


You can see that the Data Source has been already added.

The installation of our Monitoring Services are completed.
