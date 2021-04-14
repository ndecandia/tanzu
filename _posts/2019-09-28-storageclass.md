---
layout: post
title: Managing storage classes
comments: false
category: blog
---
# Managing storage classes


## Creating a storage class

```bash
kubectl create namespace nfs-provisioner
```

```bash
kubectl config set-context --current --namespace nfs-provisioner
```

> ```
> Context "kubernetes-admin@desotech" modified.
> ```

```bash
kubectl apply -f \
	https://raw.githubusercontent.com/desotech-it/DSK201-public/master/common-storageclass.yaml
```

Check the ip of NFS already provisioned:

```bash
cat ~/listaip.txt
```

Use the NFS's IP to control the exported NFS:

```bash
showmount -e 10.10.95.100
```

You should see a NFS export with this path: `/nfs/provisioner`


Now create a new file and change the IP with the NFS Server's IP and the export path of the NFS Server:

```bash
vi /home/student/nfs-pod-provisioner.yaml
```

File: `/home/student/nfs-pod-provisioner.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nfs-pod-provisioner
  name: nfs-pod-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-pod-provisioner
  template:
    metadata:
      labels:
        app: nfs-pod-provisioner
    spec:
      serviceAccountName: nfs-pod-provisioner-sa # name of service account created in rbac.yaml
      containers:
      - image: quay.io/external_storage/nfs-client-provisioner:latest
        name: nfs-client-provisioner
        resources: {}
        volumeMounts:
          - name: nfs-provisioner-v
            mountPath: /persistentvolumes
        env:
          - name: PROVISIONER_NAME # do not change
            value: nfs-deso-provisioner # SAME AS PROVISONER NAME VALUE IN STORAGECLASS
          - name: NFS_SERVER # do not change
            value: 10.10.XX.100 # Change with Ip of the NFS SERVER
          - name: NFS_PATH # do not change
            value: /nfs/provisioner  # path to nfs directory setup
      volumes:
       - name: nfs-provisioner-v # same as volumemouts name
         nfs:
           server: 10.10.XX.100 # Change with Ip of the NFS SERVER
           path: /nfs/provisioner
```

```bash
kubectl apply -f nfs-pod-provisioner.yaml
```

> ```
> deployment.apps/nfs-pod-provisioner created
> ```

```bash
vi /home/student/storageclass.yaml
```

File: `/home/student/storageclass.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storageclass # IMPORTANT pvc needs to mention this name
provisioner: nfs-deso-provisioner # name can be anything
parameters:
  archiveOnDelete: "false"
  volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f storageclass.yaml
```

> ```
> storageclass.storage.k8s.io/nfs-storageclass created
> ```

```bash
kubectl get sc
```

> ```
> NAME               PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
> nfs-storageclass   nfs-deso-provisioner   Delete          Immediate           false                  16s
> ```

```bash
kubectl create ns test-storageclass
```

> ```
> namespace/test-storageclass created
> ```

```bash
kubectl config set-context --current --namespace test-storageclass
```

> ```
> Context "kubernetes-admin@desotech" modified.
> ```

```bash
vi /home/student/pvc01.yaml
```

File: `/home/student/pvc01.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  storageClassName: nfs-storageclass
```

Create this PVC:

```bash
kubectl apply -f pvc01.yaml
```

> ```
> persistentvolumeclaim/pvc01 created
>```

```bash
kubectl get pv,pvc
```

> ```
> NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS       REASON   AGE
> persistentvolume/pvc-e62ff4ba-ac2f-4edc-b2a7-b571856739bb   1G         RWO            Delete           Bound    test-storageclass/pvc01   nfs-storageclass            16s
>
> NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
> persistentvolumeclaim/pvc01   Bound    pvc-e62ff4ba-ac2f-4edc-b2a7-b571856739bb   1G         RWO            nfs-storageclass   17s
> ```

```bash
kubectl  get sc
```

> ```
> NAME               PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
> nfs-storageclass   nfs-deso-provisioner   Delete          Immediate           false                  9m13s
> ```

```bash
vi /home/student/pvc02.yaml
```

File: `/home/student/pvc02.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc02
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
```

Create this PVC:

```bash
kubectl apply -f pvc02.yaml
```

> ```
>  persistentvolumeclaim/pvc02 created
> ```

```bash
kubectl get pvc
```

> ```
> NAME    STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
> pvc01   Bound     pvc-e62ff4ba-ac2f-4edc-b2a7-b571856739bb   1G         RWO            nfs-storageclass   2m58s
> pvc02   Pending                                                                                           18s
> ```

## Setting a default storage class

Kubernetes v1.6 added the ability to set a default storage class. This is the storage class that will be used to
provision a PV if a user does not specify one in a PVC.

You can define a default storage class by setting the annotation `storageclass.kubernetes.io/is-default-class` to
`true` in the storage class definition. According to the specification, any other value or absence of the annotation is
interpreted as `false`.

It is possible to configure an existing storage class to be the default storage class by using the following command:

```bash
kubectl patch storageclass nfs-storageclass \
	-p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```bash
kubectl get sc
```

> ```
> NAME                         PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
> nfs-storageclass (default)   nfs-deso-provisioner   Delete          Immediate           false                  11m
> ```

```bash
kubectl delete -f pvc02.yaml
```

> ```
> persistentvolumeclaim "pvc02" deleted
> ```

```bash
kubectl apply -f pvc02.yaml
```

> ```
> persistentvolumeclaim/pvc02 created
> ```

```bash
kubectl  get pvc
```

> ```
> NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
> pvc01   Bound    pvc-e62ff4ba-ac2f-4edc-b2a7-b571856739bb   1G         RWO            nfs-storageclass   4m42s
> pvc02   Bound    pvc-b9af9093-a3d6-46d4-9e84-72bc86c77c1a   1G         RWO            nfs-storageclass   3s
> ```

## Clean up

```bash
kubectl  delete ns test-storageclass
```

> ```
> namespace "test-storageclass" deleted
> ```

```bash
kubectl config set-context --current --namespace default
```

> ```
> Context "kubernetes-admin@desotech" modified.
> ```
