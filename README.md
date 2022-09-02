# Steps to reproduce: OADP demo. backup & restore Pacman application with DataMover.

**_NOTE:_** OADP version 1.1 is needed for this Demo to work.

**_WARN:_** This is just an example to show how the DataMover feature works in OADP is not official documentation of the product.

1. Deploy OCP 4.11  

2. Deploy ODF 4.11. Example. [Deploying ODF with ArgoCD](https://github.com/red-hat-storage/argocd-odf)  

3. Create the `openshift-adp` namespace and operator-group

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/01_namespace_and_operatorgroup.yaml
```

4. Deploy OADP v1.1 Operator

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/02_oadp-operator.yaml
```
    
5. Deploy the Volsync Operator

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/03_volsync-operator.yaml
```
    
6. Create the Restic repository secret that contains the repo encryption key,
by default OADP consumes a secret with the name: `dm-restic-secret`

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/04_restic-secret.yaml
```
    
7. Create a cloud-secret file with the credentials for your S3 backup target.
Our example:

```
$ cat aws-oadp-creds.txt
[default]
aws_access_key_id = AKIAVS4L7GDXXXXXX
aws_secret_access_key = 4amiqN9pw3cy/uXXXXXXXvieXqkkJgSLF1r
```
    
8. Create the cloud-secret from the previous file

```
$ oc create secret generic cloud-credentials --namespace openshift-adp --from-file cloud=aws-oadp-creds.txt
```
    
9. Create the AWS S3 target bucket that you are going to use for your OADP
backups/restores, using an S3 client and the AWS creds from the aws-oadp-creds.txt file to create
the bucket. In this example we will use oadp-store as the name for the bucket:

```
$ aws s3 mb s3://oadp-store/
```

10. Create the Data protection application. in this DPA yaml, we are using AWS
S3 us-east-1 region with a bucket called oadp-store as the storage location
for OADP, the S3 access and secret keys will be read from the cloud-credentials
secret created previously, we are also enabling the DataMover feature that is TP in OADP v1.1

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/05_dpa.yaml
```
   
11. Once the DPA is created you should have three pods in the openshift-dpa
namespace, the volume-snapshot-mover will be available only when the DataMover
the feature is enabled:

```
$ oc get pods
NAME                                                READY   STATUS    RESTARTS   AGE
openshift-adp-controller-manager-6f45cf8766-6nnhj   1/1     Running   0          14m
velero-8c6c879fc-tm9nk                              1/1     Running   0          65s
volume-snapshot-mover-78d694d4b5-7bztf              1/1     Running   0          65s
```
    
12. Also the Backup Storage Location should be in the Available state:

```
oc get backupstorageLocations
NAME          PHASE       LAST VALIDATED   AGE     DEFAULT
oadp-demo-1   Available   41s              6m18s   true
```

13. Setting a DeletionPolicy of Retain on the VolumeSnapshotClass will preserve the volume snapshot in the storage system for the lifetime of the Velero backup and will prevent the deletion of the volume snapshot

```
$ oc patch volumesnapshotclass ocs-storagecluster-rbdplugin-snapclass --type=merge -p '{"deletionPolicy": "Retain"}'
$ oc patch volumesnapshotclass ocs-storagecluster-cephfsplugin-snapclass --type=merge -p '{"deletionPolicy": "Retain"}'

$ oc label VolumeSnapshotClass  ocs-storagecluster-cephfsplugin-snapclass velero.io/csi-volumesnapshot-class=true
$ oc label VolumeSnapshotClass ocs-storagecluster-rbdplugin-snapclass velero.io/csi-volumesnapshot-class=true
```

14. Deploy the PACMAN application.

```
$ git clone git@github.com:likid0/oadp-demo.git
$ oc new-project pacman
$ oc apply -k oadp-demo/pacman/
```
   
15. Play Pacman!!! and create High Scores

16. Take a backup of Pacman, via a normal backup:

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/06_backup-pacman.yaml
```

    or scheduling a planned time for the backup:

```
$ oc create -f https://github.com/likid0/oadp-demo/blob/main/07_schedule-pacman.yaml
```

17. Confirm your backup has finished successfully without errors, for example using the velero cli

```
$ alias velerocli='oc -n openshift-adp exec deployment/velero -c velero -it -- ./velero'
$ velerocli backup get
```

18. Delete the Pacman Namespace, or drop the full OCP cluster

```
$ oc delete ns pacman
```

19. Restore the Pacman Application.

```
$ oc create -f https://raw.githubusercontent.com/likid0/oadp-demo/main/08_restore-pacman.yaml
```

20. Check the restore has finished successfully

```
$ velerocli restore get
```
