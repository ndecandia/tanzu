---
layout: post
title: Kubernetes Dashboard
comments: false
category: blog
---
# Kubernetes Dashboard


Kubernetes has an optional web-based dashboard that you can deploy to your cluster. Let’s set it up now.

We will use an insecure method of exposing the Kubernetes Dashboard to keep things simple for the lab.
In production, you should deploy the dashboard with proper authenticaion and TLS configured.

## Deploy the Dashboard

First deploy the dashboard

```bash
kubectl apply -f  https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```
> ```
> namespace/kubernetes-dashboard created
> serviceaccount/kubernetes-dashboard created
> service/kubernetes-dashboard created
> secret/kubernetes-dashboard-certs created
> secret/kubernetes-dashboard-csrf created
> secret/kubernetes-dashboard-key-holder created
> configmap/kubernetes-dashboard-settings created
> role.rbac.authorization.k8s.io/kubernetes-dashboard created
> clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
> rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
> clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
> deployment.apps/kubernetes-dashboard created
> service/dashboard-metrics-scraper created
> deployment.apps/dashboard-metrics-scraper created
> ```

To get full cluster access to the `kubernetes-dashboard` account, run the following command:

```bash
kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin \
	--serviceaccount=kubernetes-dashboard:kubernetes-dashboard
```

> ```
> clusterrolebinding.rbac.authorization.k8s.io/add-on-cluster-admin created
> ```

Then check the `kubernetes-dashboard` services and change the type for external access:

```bash
kubectl get svc -n kubernetes-dashboard
```

> ```
> NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
> dashboard-metrics-scraper   ClusterIP   10.97.164.74    <none>        8000/TCP   70s
> kubernetes-dashboard        ClusterIP   10.111.160.99   <none>        443/TCP    70s
> ```

Change the type of the service with the `edit` command, then replace the type `ClusterIP` with `LoadBalancer`:

```bash
kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard
```

from this

> ```
> type: ClusterIP
> ```

to:

> ```
> type: LoadBalancer
> ```

then save and exit. Check the services now:

```bash
kubectl get svc -n kubernetes-dashboard
```

> ```
> NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)         AGE
> dashboard-metrics-scraper   ClusterIP      10.97.164.74    <none>         8000/TCP        18m
> kubernetes-dashboard        LoadBalancer   10.111.160.99   10.10.95.202   443:31938/TCP   18m
> ```

Now connect to the external IP in HTTPS from your browser. The URL will be `https://<public-ip>`.

Since we are using a self-signed cert, you will need to advance past the browser certificate warnings to access the
dashboard. On Windows, you will need Firefox to get past this. On MacOS, you can use Chrome or Firefox.

When prompted for credentials after browsing to the Dashbaord URL, select the token option and provide the token from
`k8s-dashboard` output in the command below.

```bash
kubectl describe  -n kubernetes-dashboard secret kubernetes-dashboard-token
```

> ```
> Name:         kubernetes-dashboard-token-hwng9
> Namespace:    kubernetes-dashboard
> Labels:       <none>
> Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
>               kubernetes.io/service-account.uid: 9d80906d-2800-4682-960d-651e6352e0bc
>
> Type:  kubernetes.io/service-account-token
>
> Data
> ====
> namespace:  20 bytes
> token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlNCUHZ3V2lTRmpKQk83T3BJMG5iODdwM05yQ253cHNQUkNnMWN1eWVmdmsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1od25nOSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjlkODA5MDZkLTI4MDAtNDY4Mi05NjBkLTY1MWU2MzUyZTBiYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.FNOBqD69iV-xpydbiuwjg2ANanOhSj2ZWtzw_7S55WTwrAX2LOX4B7k59_779HH8_y1w9PDswKQKsx_oNTgorzk84lL-GtMUyPJ4ivtKFKLQBYxYgtv1PrqcqQEQ4_uandl2Jd0paJzz2cce5FtmQV7jbFDFHPH2A34y_LOu6E-aIsxK4QQRVozGHu_Vcf7fDjSvWJ2l9UlSw5KS_FoS6sFX9q1pyGcpzAAJ5kyzCE_eWwaqyPnCO1JCZUjGxp6K2PmsapI3SRlAmaZhkdzFI5EkxU2NvG7k-jmPu6h5F0QxFDcY0M1oEdc5jwmtLHIY6LDnEBENz6soe4yqSD3O0g
> ca.crt:     1025 bytes
> ```

If you are using the terminal built into the Desotech Terminal web interface, the token may be split across multiple
lines. Copy and paste the token into an editor, remove the line breaks, and then paste it into the Dashboard login.

For now, look around to ensure you can connect. Keep this URL handy as we’ll visit the dashboard in upcoming chapters
when more objects have been populated.
