---
layout: post
title: Backup and Restore
comments: false
category: blog
---
# Backup and Restore


Backing up a Kubernetes cluster enables operators to recover from cluster failures or the accidental deletion of
objects. [Velero](https://velero.io/) is a free, open-source, tool for backing up and restoring all, or portions of a
Kubernetes cluster. In this lab, we’ll install Velero, backup our Kubernetes cluster, delete a namespace, and perform a
restore.

## Installing Velero

Start by downloading and installing the Velero command line tool.

```bash
cd ~

wget https://github.com/vmware-tanzu/velero/releases/download/v1.5.2/velero-v1.5.2-linux-amd64.tar.gz

tar zxvf velero-v1.5.2-linux-amd64.tar.gz

sudo cp velero-v1.5.2-linux-amd64/velero /usr/local/bin/
```

Velero requires an object storage provider to store it’s backups. It supports many of the major cloud object storage
providers including Amazon S3 and Google GCS. For our lab, we will deploy Minio, which is an S3 compatible object store
which can be run directly in our lab environment.

```bash
kubectl apply -f \
  $HOME/velero-v1.5.2-linux-amd64/examples/minio/00-minio-deployment.yaml
```

> ```
> namespace/velero created
> deployment.apps/minio created
> service/minio created
> job.batch/minio-setup created
> ```

Next, create a credentials file that Velero will use to authenticate with Minio.

> Use the key_id and access_key exactly as shown below.
> These are the default credentials for Minio.
> While Minio does allow you to change the credentials, we’ll use the default ones to keep things simple for now.

```bash
cd velero-v1.5.1-linux-amd64
```

> ```
> cat <<EOF >>credentials-velero
> [default]
> aws_access_key_id = minio
> aws_secret_access_key = minio123
> EOF
> ```

Now use the Velero command line tool to deploy the Velero server to the Kubernetes cluster.

```bash
velero install \
	--provider aws \
	--bucket velero \
	--secret-file ./credentials-velero \
	--use-volume-snapshots=false \
	--plugins=velero/velero-plugin-for-aws:v1.1.0 \
	--backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

> ```
> CustomResourceDefinition/backups.velero.io: attempting to create resource
> CustomResourceDefinition/backups.velero.io: created
> CustomResourceDefinition/backupstoragelocations.velero.io: attempting to create resource
> CustomResourceDefinition/backupstoragelocations.velero.io: created
> CustomResourceDefinition/deletebackuprequests.velero.io: attempting to create resource
> CustomResourceDefinition/deletebackuprequests.velero.io: created
> CustomResourceDefinition/downloadrequests.velero.io: attempting to create resource
> CustomResourceDefinition/downloadrequests.velero.io: created
> CustomResourceDefinition/podvolumebackups.velero.io: attempting to create resource
> CustomResourceDefinition/podvolumebackups.velero.io: created
> CustomResourceDefinition/podvolumerestores.velero.io: attempting to create resource
> CustomResourceDefinition/podvolumerestores.velero.io: created
> CustomResourceDefinition/resticrepositories.velero.io: attempting to create resource
> CustomResourceDefinition/resticrepositories.velero.io: created
> CustomResourceDefinition/restores.velero.io: attempting to create resource
> CustomResourceDefinition/restores.velero.io: created
> CustomResourceDefinition/schedules.velero.io: attempting to create resource
> CustomResourceDefinition/schedules.velero.io: created
> CustomResourceDefinition/serverstatusrequests.velero.io: attempting to create resource
> CustomResourceDefinition/serverstatusrequests.velero.io: created
> CustomResourceDefinition/volumesnapshotlocations.velero.io: attempting to create resource
> CustomResourceDefinition/volumesnapshotlocations.velero.io: created
> Waiting for resources to be ready in cluster...
> Namespace/velero: attempting to create resource
> Namespace/velero: already exists, proceeding
> Namespace/velero: created
> ClusterRoleBinding/velero: attempting to create resource
> ClusterRoleBinding/velero: already exists, proceeding
> ClusterRoleBinding/velero: created
> ServiceAccount/velero: attempting to create resource
> ServiceAccount/velero: created
> Secret/cloud-credentials: attempting to create resource
> Secret/cloud-credentials: created
> BackupStorageLocation/default: attempting to create resource
> BackupStorageLocation/default: created
> Deployment/velero: attempting to create resource
> Deployment/velero: created
> Velero is installed! ⛵ Use 'kubectl logs deployment/velero -n velero' to view the status.
> ```


## Creating a Backup

Velero can backup an entire cluster or just certain objects. We’ll backup the entire cluster now.

```bash
velero backup create my-cluster-backup
```

> ```
> Backup request "my-cluster-backup" submitted successfully.
> Run `velero backup describe my-cluster-backup` or `velero backup logs my-cluster-backup` for more details.
> ```

Check the status of the backup by running the following command:

```bash
velero backup describe my-cluster-backup
```

> ```
> Name:         my-cluster-backup
> Namespace:    velero
> Labels:       velero.io/storage-location=default
> Annotations:  velero.io/source-cluster-k8s-gitversion=v1.19.2
>               velero.io/source-cluster-k8s-major-version=1
>               velero.io/source-cluster-k8s-minor-version=19
>
> Phase:  Completed
>
> Errors:    0
> Warnings:  0
>
> Namespaces:
>   Included:  *
>   Excluded:  <none>
>
> Resources:
>   Included:        *
>   Excluded:        <none>
>   Cluster-scoped:  auto
>
> Label selector:  <none>
>
> Storage Location:  default
>
> Velero-Native Snapshot PVs:  auto
>
> TTL:  720h0m0s
>
> Hooks:  <none>
>
> Backup Format Version:  1.1.0
>
> Started:    2020-09-25 19:53:44 +0200 CEST
> Completed:  2020-09-25 19:53:53 +0200 CEST
>
> Expiration:  2020-10-25 18:53:44 +0100 CET
>
> Total items to be backed up:  680
> Items backed up:              680
>
> Velero-Native Snapshots: <none included>
> ```

When the backup is ready, `Phase` will show as `Completed`.

In this case, we manually initiated a backup, but Velero also supports scheduling backups.

```bash
velero get backups
```

> ```
> NAME                STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
> my-cluster-backup   Completed   0        0          2020-09-25 19:53:44 +0200 CEST   29d       default            <none>
> ```

## Restoring from a Backup

Now let’s simulate a common scenario which would require a restore. Delete the `logging` namespace.
This will not only delete the namespace, but all the pods and services we deployed within it.

```bash
kubectl delete ns logging
```

> ```
> namespace "logging" deleted
> ```

> The namespace can take up to a minute to delete. The terminal will appear to hang during this time.

Verify that the namespace and pods within have been deleted.

```bash
kubectl get ns
```

> ```
> NAME              STATUS   AGE
> default           Active   239d
> kube-node-lease   Active   239d
> kube-public       Active   239d
> kube-system       Active   239d
> metallb-system    Active   239d
> monitoring        Active   4h21m
> nfs-provisioner   Active   33h
> velero            Active   16m
> ```

```bash
kubectl get pods -n logging
```

> ```
> No resources found in logging namespace.
> ```

Now let’s use Velero to restore the deleted namespace.

```bash
velero restore create --from-backup my-cluster-backup
```

> ```
> Restore request "my-cluster-backup-20200925200659" submitted successfully.
> Run `velero restore describe my-cluster-backup-20200925200659` or `velero restore logs my-cluster-backup-20200925200659` for more details.
> ```

Check the progress of the restore operation by running the `velero restore describe` command from the output of the
last command.

> You can safely ignore any warnings in the output.

```bash
velero restore describe my-cluster-backup-20200925200659
```

When `Phase` shows as `Completed`, the restore is complete.
Verify that the namespace and pods within have been restored.

```bash
velero get restores
```

> ```
> NAME                               BACKUP              STATUS      STARTED                          COMPLETED                        ERRORS   WARNINGS   CREATED                          SELECTOR
> my-cluster-backup-20200925200659   my-cluster-backup   Completed   2020-09-25 20:06:59 +0200 CEST   2020-09-25 20:07:56 +0200 CEST   0        69         2020-09-25 20:06:59 +0200 CEST   <none>
> ```

```bash
kubectl get ns
```

> ```
> NAME              STATUS   AGE
> default           Active   239d
> kube-node-lease   Active   239d
> kube-public       Active   239d
> kube-system       Active   239d
> logging           Active   3m57s
> metallb-system    Active   239d
> monitoring        Active   4h26m
> nfs-provisioner   Active   33h
> velero            Active   20m
> ```

```bash
kubectl get pods -n logging
```

> ```
> NAME                             READY   STATUS    RESTARTS   AGE
> elasticsearch-master-0           1/1     Running   0          4m
> elasticsearch-master-1           1/1     Running   0          4m
> elasticsearch-master-2           1/1     Running   0          4m
> filebeat-filebeat-6gpvp          1/1     Running   0          4m
> filebeat-filebeat-7dtvs          1/1     Running   0          4m
> filebeat-filebeat-cpbfk          1/1     Running   0          4m
> kibana-kibana-5c85d47cff-4md2n   1/1     Running   0          3m59s
> ```

This lab demonstrated a fairly simple backup and recovery using Velero.
Velero can backup and restore a subset of the cluster as demonstrated in this lab, but can also be used to restore an
entire cluster in the event of a major failure. You also ran a single manual backup, but Velero can also be configured
to run backups on a schedule.
