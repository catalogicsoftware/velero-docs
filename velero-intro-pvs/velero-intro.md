# Using Velero with Persistent Volumes

*Raghuram Devarakonda, Chief Architect, CloudCasa*

*March 15, 2023*

Note:
The contents of this file are meant to be presented as slides using
reveal.js (https://github.com/hakimel/reveal.js/) framework. 

---

## Introduction

- Velero is a very popular open source Kubernetes backup tool.

- It supports many use cases such as cluster migration and disaster recovery.

---

## Types of Backups

- Resource data

- PV Snapshots

  - CSI
  
  - Native snapshots. E.g. EBS, Google persistent disks, Azure disks.

- PV backups (data transfer)

  - Restic, Kopia

---

## Object stores

- AWS S3, Azure blob storage, Google storage

- Any other S3 compatible storage.

Note:

It is possible to add support for an object storage provider by implementing a plug-in.

---

## Installation

- Different methods of installation are available.

  - CLI, Helm etc.
  
- To use CLI method, download *velero* from github page.

  https://github.com/vmware-tanzu/velero/releases

Note:

In this presentation, Velero 1.10.1 will be used.

---

## Installation - Contd

- Where will the backups be stored?

- Snapshots

  - Do you want to enable CSI snapshots?
  
  - Are there non-CSI PVs that need to be snapshotted?
  
- Do you want to enable file system backups?

  - If so, Restic or Kopia?

Note:

In our case, backups will be stored in an AWS bucket (authenticated using access/secret keys). We will enable CSI
snapshots but will not work with any non-CSI PVs. We will also see how file system backups can be done using Restic.

One important point to note is that many articles on Velero, including Velero's official docs, combine installation with
objectstore configuration. It is always better to keep installation separate from configuration so in the following
command, we will only install Velero components.

We will be using a local minikube cluster for the demo, including for CSI features.

``` bash

# Optional: Create minikube cluster called "velero" to be used for the demo.
$ minikube -p velero start

$ velero install \
    --plugins velero/velero-plugin-for-aws:v1.6.1,velero/velero-plugin-for-csi:v0.4.2 \
    --use-node-agent --features=enableCSI \
    --no-default-backup-location --no-secret --use-volume-snapshots=false
```

By default, Velero components are installed in the namespace *Velero* but it is possible to install in any other
custom namespace by passing the value with the option *--namespace*. 

---

## Configure backup storage

- The resource *BackupStorageLocation* encapsulates backup storage which is typically S3 compatible bucket (or Azure
  blob storage). 

- It can be directly created using *kubectl* or by using *velero* CLI.

Note:

We will use AWS S3 bucket and use access key/secret key method for authentication. So a secret needs to be created first
with the credentials. Create a file called "aws-creds" in the following format::

    [default]
    aws_access_key_id=<ACCESS_KEY>
    aws_secret_access_key=<SECRET_KEY>

``` bash
$ kubectl -n velero create secret generic aws-creds --from-file=aws=aws-creds

$ velero backup-location create --bucket velero-demo --credential aws-creds=aws \
    --provider aws --config region=us-east-1 aws-velero-intro

# List BSLs.
$ velero backup-location get

# Similar but with kubectl
$ kubectl -n velero get bsl

```

---

## Basic resource backup

- Velero includes the specs of all the selected resources in the backup in the form of a tar file. 

- Resources can be selected by namespaces, resource types, and labels. 

Note:

``` bash
$ velero backup create --storage-location aws-velero-intro basic-backup

# To see the status of the backup
$ velero backup get basic-backup

# To see detailed status of the backup
$ velero backup describe basic-backup

# To see list of resources in the backup
$ velero backup describe --details basic-backup

# To see the logs of the backup
$ velero backup logs basic-backup

```

Velero creates hierarchy of files on the target storage. Here is what you will see after above backup:

```bash
    29 backups/basic-backup/basic-backup-csi-volumesnapshotclasses.json.gz
    29 backups/basic-backup/basic-backup-csi-volumesnapshotcontents.json.gz
    29 backups/basic-backup/basic-backup-csi-volumesnapshots.json.gz
 15931 backups/basic-backup/basic-backup-logs.gz
    29 backups/basic-backup/basic-backup-podvolumebackups.json.gz
  2657 backups/basic-backup/basic-backup-resource-list.json.gz
    29 backups/basic-backup/basic-backup-volumesnapshots.json.gz
112790 backups/basic-backup/basic-backup.tar.gz
  2557 backups/basic-backup/velero-backup.json
```
---

## CSI

- Container Storage Interface (CSI) is a standard to expose storage systems to containers. 

- With CSI support, Kubernetes allows containers to request storage using ``PersistenVolumeClaims``.

- Recent CSI versions also support snapshotting persistent volumes.

  - Not all CSI drivers support CSI functionality. E.g. EFS CSI driver.

---

## CSI Configuration

- Since CSI is not part of core Kubernetes, it is not installed by default.

- Extra components need to be installed to be able to use CSI.

  - CRDs, External snapshotter, CSI driver.
  
  - Cloud providers make the process a bit easier compared to on-prem distributions.

---

## CSI Configuration - Contd

- If the CSI configuration is not done correctly, snapshots will not work.

- We see many CloudCasa users as well as Velero users complaining about snapshots not working and in most cases, the
  problem is with the CSI configuration.
  
- CloudCasa developed a script that can check if your CSI configuration is valid.

  https://docs.cloudcasa.io/help/kbs/kb-csi-checker.html

Note:

```bash

# Using the CSI verification script from CloudCasa.

$ curl -s https://raw.githubusercontent.com/catalogicsoftware/cloudcasa-artifacts/master/scripts/cloudcasa-csi-checker.sh | bash 2>&1 - | tee storage.txt

# To enable CSI on the minikube cluster

$ minikube -p velero addons enable volumesnapshots

$ minikube -p velero addons enable csi-hostpath-driver

```

---

## CSI with Velero

- Velero supports taking snapshots of CSI volumes.
  - For each CSI driver, you need to have a *VolumeSnapshotClass* with the deletion policy set to "Retain". 
  - Set label ``velero.io/csi-volumesnapshot-class=true``.

- It is also possible to "restore" by creating PVCs from CSI snapshots.

Note:

``` bash

# create a CSI PVC and a Pod with that PVC.
$ kubectl apply -f csi-pod.yaml

# Confirm that pod is running
$ kubectl -n testapp-csi get pod

# Create some test data
$ kubectl -n testapp-csi exec -it testapp sh

# Inside the container shell
$ date > /data/testfile

# Label volume snapshot class so that Velero knows which one to pick.
$ kubectl label volumesnapshotclass csi-hostpath-snapclass velero.io/csi-volumesnapshot-class=true

# Create CSI backup.
$ velero backup create --storage-location aws-velero-intro csi-backup

# Check backup progress
$ velero backup get csi-backup

# After backup is done, confirm that snapshots are created.
$ kubectl get -A volumesnapshot
$ kubectl get volumesnapshotcontent

# Restore the namespace "testapp-csi" to a new namespace "restore-csi".
$ velero restore create csi-restore --from-backup csi-backup \
    --include-namespaces testapp-csi --namespace-mappings testapp-csi:restore-csi

# Check restore progress
$ velero restore get csi-restore

# Confirm that you see the namespace "restore-csi"
$ kubectl get ns

# Confirm that PVC is created from CSI snapshot. You should see the CSI snapshot
# under "dataSource" element.
$ kubectl -n restore-csi get pvc -o yaml

# Get a shell inside the restored Pod
$ kubectl -n restore-csi get pod
$ kubectl -n restore-csi exec -it testapp sh

# Inside the container
$ ls /data # Should show "testfile"

# Should show a timestamp verifying that backup data was restored successfully.
$ cat /data/testfile 

```

---

## File system backups

- File system backups read files in PVs and transfer them to object storage.

- Velero can use either Restic or Kopia for data transfer.

  - Both support compression, deduplication, and encryption though there are some caveats. 

- File system backups are only way for PVs that do not support snapshots. E.g NFS and EFS.

- Snapshot and file system backups cannot be combined.

Note:

```bash
# Create a backup that will use Restic to backup PVs.
$ velero backup create --storage-location aws-velero-intro \
        --default-volumes-to-fs-backup=true fs-backup
        
# Check backup progress
$ velero backup get fs-backup
```

You can see "restic" objects created in the bucket, like so:

``` bash
     29 backups/fs-backup/fs-backup-csi-volumesnapshotclasses.json.gz
     29 backups/fs-backup/fs-backup-csi-volumesnapshotcontents.json.gz
     29 backups/fs-backup/fs-backup-csi-volumesnapshots.json.gz
  17947 backups/fs-backup/fs-backup-logs.gz
   1365 backups/fs-backup/fs-backup-podvolumebackups.json.gz
   3608 backups/fs-backup/fs-backup-resource-list.json.gz
     29 backups/fs-backup/fs-backup-volumesnapshots.json.gz
 127459 backups/fs-backup/fs-backup.tar.gz
   2548 backups/fs-backup/velero-backup.json
    155 restic/testapp-csi/config
    463 restic/testapp-csi/keys/7f4b3e4cc858e4e87262335818963026eeb3b684c25f0aef22f55a2d0ab69b97
    155 restic/velero/config
    245 restic/velero/data/03/031a758d720ebd9ff0ff7a536d51eda6809bd521dbde4cd13e1ec2ca076af819
   3137 restic/velero/data/94/9413a42cf44cbbf9847e80b6d82d02804cbe69f3206a8f09f7cd2ac4e9420325
   ...
   ...
```

``` bash
# Restore the namespace "testapp-csi" to a new namespace "restore-fs".
$ velero restore create fs-restore --from-backup fs-backup \
    --include-namespaces testapp-csi --namespace-mappings testapp-csi:restore-fs

# Check restore progress
$ velero restore get fs-restore

# Confirm that you see the namespace "restore-fs"
$ kubectl get ns

# Confirm that PVC is created and Pod is running.
$ kubectl -n restore-fs get pvc
$ kubectl -n restore-fs get pod

# Get a shell inside the restored Pod
$ kubectl -n restore-fs exec -it testapp sh

# Inside the container
$ ls /data # Should show "testfile"
# Should show a timestamp verifying that backup data was restored successfully.
$ cat /data/testfile

```

---

## File system backups - Contd

- Data is read from PVs directly so backups may not be consistent. 

- It is highly advisable to use Velero hooks to freeze and thaw applications.

---

## Velero - Limitations

- Velero is a single cluster solution so to manage multiple clusters, you may need to build some tooling on top of
  Velero. 
  
- Backups cannot be done from snapshots.

---

## Resources

- https://velero.io/docs/v1.10/

- https://restic.net/

- https://kopia.io/

- https://github.com/kubernetes-csi/external-snapshotter

---

## Conclusion

- Velero is a popular open source option to protect data in a Kubernetes cluster. 

- But you may need to implement few scripts/tools to effectively manage multiple clusters.

---
