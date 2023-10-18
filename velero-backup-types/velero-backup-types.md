# Velero backup types

*Raghuram Devarakonda, Chief Architect, CloudCasa*

*October 18, 2023*

Note:
The contents of this file are meant to be presented as slides using
reveal.js (https://github.com/hakimel/reveal.js/) framework. 

---

## Agenda

- Introduction

- Snapshots

- File system backups

- Snapshot data mover backups

---

## Introduction

- Velero is a very popular open source Kubernetes backup tool.

  - It supports many use cases such as cluster migration and disaster recovery.
  
- Velero supports backup of resources as well as backup of PV data.  

---

## Basic resource backups

- Includes resource specs for all the selected resources.
  - Use the option "--include-cluster-resources" or "--include-cluster-scoped-resources" to include cluster resources in
    the backup.

- Selection can be done in different ways

  - Namespaces (include/exclude)
  - Labels
  - Resource types. E.g. pods, pvs, pvcs
  
- PV data is not backed up.

Note:

Let us start with the most basic backup supported by Velero and that is backup of resource specs. You can select
namespaces, resource types, or by labels and Velero is going read the spec of all supported resources and builds tar
files and uploads them to the objectstore. Think of this as kind of etcd backup but not exactly. Since Velero iterates
over each resource and resources can be getting updated as the backup is going on, the backup may not be "consistent"
but in practice, this may not be a big issue as you usually pick a backup window when the cluster is less active. Note
that in this backup, data in PVs is not included. This is only resource spec backup. Also note that resource backup is
the baseline and is going to be part of every other backup that we will see shortly.

```bash
velero backup create --storage-location do-bucket --snapshot-volumes=false basic-backup
```

---

## Backup of PV data

- Snapshots

- File system backups

- Snapshot data mover backups

Note:

Let us now move onto PV backup and this is where you will start seeing multiple options. Until recently, Kubernetes was
thought of a great way to run stateless apps without any need to protect data in PVs but that has changed. You can see
that more and more people are running stateful workloads in Kubernetes in production and that means, you need to backup
PV data. For this, there are multiple options but let us start with snapshots.

---

## Snapshots

- CSI Snapshots

- Provider snapshots

Note:

CSI is the container storage interface which is the storage spec that is standardized on Kubernetes and by now, most
people only use CSI type PVs. 

Provider snapshots use the APIs of the underlying storage provider to take snapshots.

---

## CSI Snapshots

- Installation of CSI driver and external snapshotter.
  - Easier to do on cloud providers where they usually provide a CSI "add-on".
  
- To enable Velero CSI functionality, pass "--EnableCSI" option to velero install command and also include CSI plug-in.

- Make sure you use right CSI plugin version. 

  - Velero 1.12: v0.6.0
  - Velero 1.11: v0.5.1

---

## CSI Snapshots - contd

- You need to have volumesnapshotclass resource for each CSI driver 
  - Set the label ``velero.io/csi-volumesnapshot-class: "true"``.

- To create PV snapshot, Velero creates "VolumeSnapshot" resource.

- If CSI is not properly configured, snapshot will not be created and Velero times out after 10 minutes.

Note:

The most prevalent use case is snapshots of CSI PVs. For this,
velero utilizes CSI snapshot interface which does mean that you need to get your CSI configuration right. Unfortunately,
this is easier said than done. You will need to install some additional components such as external snapshotter in
addition to the CSI driver itself. You need to pay attention to the version of the external snapshotter. Many times, we
see errors where Velero creates a VolumeSnapshot resource - which is like a snapshot request, but snapshot is not
created. Velero waits for 10 minutes and moves on, marking this PV as failed. This problem usually happens either
because external snapshotter is not there or is there but the version present is not compatible with CSI driver. I
highly recommend that you verify CSI snapshot work independently before doing Velero CSI snapshots. You can use the
following script from CloudCasa to verify it automatically:

https://docs.cloudcasa.io/help/kbs/kb-csi-checker.html


``` bash
velero backup create --storage-location do-bucket --snapshot-volumes=true csi-backup
```

At the end of backup, you should find cluster scoped VolumeSnapshotContent resources. 

---

## Provider snapshots

- PVs using non-CSI drivers. E.g. EBS.

- Snapshots are taken using provider specific APIs.

Note:

CSI is the future of storage in Kubernetes but there may still be clusters out there using non-CSI PVs - such as
EBS. For such PVs, Velero supports native provider snapshots. So instead of CSI snapshot, cloud provider native snapshot
will be taken.

---

## Snapshot limitations

- Snapshots are not backups.

- They are tied to primary storage and you can lose them along with the primary storage itself.

  - Even if snapshots result in data transfer, they are still tied to source. E.g, region.

- Still useful to keep last few snapshots for quick restores.

Note:

Which ever snapshots are used, I want to highlight the fact that snapshots are not backups. 
Snapshots typically are in the same primary storage where your application data itself resides. So if there is an issue
with primary storage, you may lose snapshots as well. In some cases, snapshots may indeed transfer data. For example,
snapshot of EBS PV will result in data transfer to an S3 bucket. These are some times called "durable" snapshots. Even
in this case, there will be restrictions in how you can restore. For example, you won't be able to restore to a
different region from where the source is located. You can certainly keep few snapshots around for quick restore in some
limited cases but it cannot be your only backup strategy. 

---

## File system backup (FSB)

- FSB backs up PV data to object store using "Restic" or "Kopia".

  - Must pass "--use-node-agent" install option.

  - Restic or Kopia can be selected using "--uploader-type" install option. Default is "Kopia".

- One backup repository created per namespace. 

Note:

That brings us to next backup type known as "File system backup" or FSB. To make use of these backups, you will need to
install "nodeagent" component. This is DaemonSet that runs a pod on each node in the
cluster. The main idea behind this backup is to read the data from PV and back it up to object storage using either Restic
or Kopia. These are independent tools from Velero and in fact, you can use them on any
files and directories you have to back them up to object storage such as S3. Velero simply calls them on the PVs. 

Command to create FSB:

``` bash
velero backup create --storage-location velero-demo --default-volumes-to-fs-backup=true fs-backup
```

---

## FSB - Contd
  
- Both Restic and Kopia support dedup, compression. and encryption.

- Encryption uses a hard-coded password set in the secret "velero-repo-credentials". 
  
  - It can be updated but must be done before Kopia/Restic repo is created.

---

## FSB - Contd

- Reads data from live PV 

  - May result in "inconsistent" backups if the data is constantly changing while the backup goes on.
 
- Important to use Velero hooks to quiesce application and disk writes.
 
Note:

It is
very important to understand how Velero reads data from the PVs. Data is read from "live" PV so if the data is changing
actively as the backup is going on, backup will not be consistent. So it is very important that you use Velero hooks to
quiesce the disk write activity in the PV before backup is started. If you are backing up application data, it is best
to put applications in what is known as "backup mode". This can be achieved using Velero hooks as well. 

Command to see details about all the backup repositories:

``` bash
velero repo get
```

But in general, this backup should be avoided in preference to the next backup type we will discuss. But there are some
cases where this is your only option and we will discuss them shortly.

---

## Snapshot data mover backups - SDMB

- Introduced in Velero 1.12.

- It is very similar to FSB in the way it backups data to object store using Kopia. 

- Main difference is that it uses a CSI snapshot as backup source, instead of live PV.

Note:

We finally reach the last backup type and perhaps the most important. It is only introduced in Velero 1.12 so it is
fairly new. It doesn't have a catchy name yet like FSB. Sometimes it is referred to as "Snapshot data mover backup" so
for the 
purpose of this discussion, let us call it SDMB. 

``` bash
velero backup create --storage-location velero-demo --snapshot-move-data sdm-backup
```

---

## SDMB - Implementation

- Create CSI snapshot

- Create a temporary PVC with this snapshot as source.

- Backup up temporary PVC using Kopia.

- Delete temporary snapshot.

Note:

In this backup, Velero takes CSI snapshot of a PV and then creates
a temporary PVC from the snapshot. This temporary PVC is then used as the source of backup. From this point, it works
very  similar to FSB in that data is backed up using Kopia to the objectstore. Once backup is done, temporarily PVC is
deleted as is the CSI snapshot. 

---

## SDMB - Contd

- The biggest advantage of this backup over FSB is that the backup will be relatively more consistent.

  - It is still a good idea to use hooks to quiesce disk and application writes.
  
- Limitations
  
  - Only works for CSI drivers that support snapshots and restore from snapshots.

  - Only Kopia is supported.

---

## SDMB vs FSB

- Use SDMB wherever you can.

- Go with FSB in case of

  - non-CSI PVs
  
  - CSI driver doesn't support snapshots or restore from snapshots.
  
  - Need to use Restic

Note:

The biggest advantage of this backup is that the data is going to be much more consistent because Velero reads data from
a snapshot rather than from a live PV. Note that this type only works with CSI PVs and also, only Kopia is
supported. But if you are ok with these two things, you should always prefer this backup type to any other type. If your
PVCs are not CSI or you want to use Restic for some reason, your only option for backups is FSB. So the recommendation is
use SDM where you can, use FSB otherwise.

---

## CSI Snapshots - discussion

- Not all CSI drivers support snapshot interface. E.g. EFS.

- Some CSI drivers support snapshots but not restore from snapshot. E.g. Azure Files.

  - Longhorn supports creation of PVC from snapshot but behind the scenes, it actually copies the data.
  
  - This may take long time depending on data size, thus resulting in timeouts.
  
Note:

I would like to talk a bit more about CSI snapshots at this point. Not all CSI drivers support snapshots. For example,
AWS's EFS CSI driver doesn't support snapshots. In such cases, FSB is the only option. In other cases, CSI driver may
support snapshots but it may not support restore from snapshot or restore is not instantaneous. For example, Azure files
CSI driver supports CSI snapshots but you cannot restore from it by creating a PVC using snapshot as data source. So you
won't be able to use SDMB. In some cases such as Longhorn, you can create a PVC from snapshot but behind the scenes,
Longhorn is actually going to copy data from snapshot to the new PVC. As can be expected, this is going to take time and
depending on data size, PVC may not be ready in time and Velero may time out. We have implemented SDMB like backup in
CloudCasa almost 3 years back so we have lot of experience with various issues people may run into. Since this is only
introduced in Velero recently, we fully expect lot of questions to come from community in this regard and we will help
wherever we can. 

---

## Miscellaneous

- If you are selecting specific resources types for backup and are including pvcs and pvs, you must include pods. 

  - For the same reason, FSB/SDMB won't work for unmounted PVCs.

- With Longhorn CSI driver, volume snapshot class should have the property "type: snap".

- Change global repo password immediately after installing Velero.

- Use Kopia if you can. Restic may not see many enhancements.

---

## Conclusion

- Snapshots are not backups.

- Use SDMB wherever you can. Use FSB otherwise.

---

## Questions?



