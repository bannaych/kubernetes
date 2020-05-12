# Oracle Containers on Kubernetes and Pure Storage PSO

PSO allow for smart provisioning of storage to one or many arrays, this provides administrator and developers the ability to  provision storage on demand with cloud like functionality, ease and flexibility.

In this blog I will use Pure Storage Orchestrator ( PSO ) to automate the provisioning of volumes to Kubernetes which will then be used to create Oracle containers using persistent storage.  

My next blog will demonstrate the  power or Pure Storage snapshots and cloning to rapidly create instant copies of the source Oracle database.

In my first blog https://bannaych.blogspot.com/2020/04/mssql-server-on-docker-with-pure.html I showed you how we can create SQL Server containers using the Pure Storage Docker plugin. In this blog I will show you how can leverage the Pure Storage Docker plugin to instantly create a clone volume from the source volume, then use Pure Storage crash consistent snapshots to refresh the clone volume.

Crash consistent snapshots provide us with a point in time copy of the volume/database, because we use metadata pointers to the original blocks on disk, the snapshots are created instantly with no initial space consumption, then those snapshots can be used to refresh volumes or create new ones.

The big advantage of using snapshots is that we no longer need to backup and restore large databases to provision and refresh TEST/DEV/UAT environments. In the world of DevOps being agile is imperative and Pure Storage space efficient snapshots are key to maximising productivity of database administrators and developers.
# Environment

|Role|FQDN|IP|OS|RAM|CPU|K8 Ver
|----|----|----|----|----|----|---|
|Master|master.localdomain.com|192.168.111.231|Ubuntu 18.04|5G|3|1.18
|Worker1|worker1.localdomain.com|192.168.111.232|Ubuntu 18.047|8G|4|1.18

```
# kubectl get nodes -o wide
NAME      STATUS   ROLES    AGE    VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master    Ready    master   4d4h   v1.18.2   192.168.111.231   <none>        Ubuntu 18.04.1 LTS   4.15.0-29-generic   docker://19.3.6
worker1   Ready    <none>   4d4h   v1.18.2   192.168.111.232   <none>        Ubuntu 18.04.1 LTS   4.15.0-99-generic   docker://19.3.6
```
# Install PSO

- Install helm 3 kubernetes package manager
```
# curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
# ./get_helm.sh

```
- Add the Pure Storage Repo
```
# helm repo add pure https://purestorage.github.io/helm-charts
# helm repo update

-- look for the new pure csi driver --

# helm search repo pure-csi
NAME         	CHART VERSION	APP VERSION	DESCRIPTION
pure/pure-csi	1.2.0        	1.2.0      	A Helm chart for Pure Service Orchestrator CSI ...

You can see we are using PSO version 1.2 or 5.2 
```
- Download the Pure Storage values file which is used to configure connectivity to the FlashArray or FlashBlades
```
wget https://raw.githubusercontent.com/purestorage/helm-charts/master/pure-csi/values.yaml
```
- Edit the values file with the FlashArray details
```
arrays:
  FlashArrays:
    - MgmtEndPoint: "192.168.111.130"
      APIToken: "476d58bc-3c91-0d10-1b9e-0f31058c4621"
```
- Create the PSO namespace ( Required for Helm 3 )
```
# kubectl create namespace pso
namespace/pso created
```

- Now we install PSO using the values file we created
```
# helm install pro-csi pure/pure-csi -f values.yaml --namespace pso
NAME: pso-csi
LAST DEPLOYEd: Fri 8 May 22:14:50 AEST 2020
NAMESPACE: pso
STATUS: deployed
REVISION: 1
TEST SUITE: none
```

- Confirm the new Pure Storage classes have been configured
```
# kubectl get sc
NAME         PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
pure         pure-csi      Delete          Immediate           true                   4d4h
pure-block   pure-csi      Delete          Immediate           true                   4d4h
pure-file    pure-csi      Delete          Immediate           true                   4d4h

"the pure driver will be deprecated, so use the pure-block driver"

```
- Check for all the service running in the PSO namespace

```
# kubectl get all -n pso
NAME                     READY   STATUS    RESTARTS   AGE
pod/pure-csi-864dm       3/3     Running   0          3d6h
pod/pure-provisioner-0   4/4     Running   0          3d6h

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service/pro-csi-pure-csi   ClusterIP   10.96.217.49   <none>        12345/TCP   4d4h
service/pure-provisioner   ClusterIP   10.96.125.41   <none>        12345/TCP   4d4h

NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/pure-csi   1         1         1       1            1           <none>          4d4h

NAME                                READY   AGE
statefulset.apps/pure-provisioner   1/1     4d4h
```


# Configure the Oracle 12c Deployment 


- Create the persistent volumes for Oracle, is this example I will create a one volume.
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: oravol
  labels:
    app: oracle12c
spec:
 accessModes:
 - ReadWriteOnce
 resources:
  requests:
   storage: 20Gi
 storageClassName: pure-block
 
 # kubectl create -f oravol.yaml
 persistentvolumeclaim/oravol created
```
- Confirm the PVC has been created
```
kubectl get pvc|grep oravol
oravol    Bound    pvc-5601093c-b828-4255-8a85-5fe1aed2d166   20Gi       RWO            pure-block     3d15h

kubectl get pv|grep oravol
pvc-5601093c-b828-4255-8a85-5fe1aed2d166   20Gi       RWO            Delete           Bound    default/oravol    pure-block              3d15h

```

- Check the volume has been created on the FlashArray

 
 NOTE: In this example I have not created an Oracle namespace, however in production evviroments it'd good practive
 to create a namespace for each application
 
- Create Kubernets secret to include the docker login information
```
kubectl create secret docker-registry oracle --docker-server=docker.io --docker-username=bannaych --docker-password=PASSWORD --docker-email=bannaych@gmail.com 
```

In this example I am not using a configmap file. I have includes the oracle variables inside the main yaml file, however you can create
a properties file if you prefer.

- Create and Start the database
```
kubectl apply -f oracle.yaml
deployment.apps/database configured
```
```
# kubectl get pods --all-namespaces
NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE
default                database-67d9b8bd67-hnrtl                    1/1     Running   0          2d10h
kube-system            coredns-66bff467f8-hj6gp                     1/1     Running   0          4d16h
kube-system            coredns-66bff467f8-sjqqg                     1/1     Running   0          4d16h
kube-system            etcd-master                                  1/1     Running   0          4d16h
kube-system            kube-apiserver-master                        1/1     Running   0          4d16h
kube-system            kube-controller-manager-master               1/1     Running   0          4d16h
kube-system            kube-flannel-ds-amd64-xtjg8                  1/1     Running   0          4d16h
kube-system            kube-flannel-ds-amd64-zppzb                  1/1     Running   2          4d16h
kube-system            kube-proxy-jtv9v                             1/1     Running   2          4d16h
kube-system            kube-proxy-ztq2w                             1/1     Running   0          4d16h
kube-system            kube-scheduler-master                        1/1     Running   0          4d16h
kube-system            kubernetes-dashboard-975499656-lx2tt         1/1     Running   0          2d11h
kubernetes-dashboard   dashboard-metrics-scraper-6b4884c9d5-bnn9x   1/1     Running   0          2d10h
kubernetes-dashboard   kubernetes-dashboard-7b544877d5-rmq77        1/1     Running   0          2d10h
pso                    pure-csi-864dm                               3/3     Running   0          3d18h
pso                    pure-provisioner-0                           4/4     Running   0          3d18h
```

- Check the status of the Oracle build
```
# kubectl logs database-67d9b8bd67-hnrtl

Starting /u01/app/oracle/product/12.2.0/dbhome_1/bin/tnslsnr: please wait...

TNSLSNR for Linux: Version 12.2.0.1.0 - Production
System parameter file is /u01/app/oracle/product/12.2.0/dbhome_1/admin/ORCL/listener.ora
Log messages written to /u01/app/oracle/diag/tnslsnr/database-7544cd4df7-d4hm4/listener/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=0.0.0.0)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 12.2.0.1.0 - Production
Start Date                07-MAY-2020 01:30:57
Uptime                    0 days 0 hr. 0 min. 0 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/12.2.0/dbhome_1/admin/ORCL/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/d kubectl logs database-67d9b8bd67-hnrtl/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
The listener supports no services
The command completed successfully

DONE!
```

-  Log onto the worker node

```
docker ps
CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS              PORTS               NAMES
f4c2a4904845        12a359cd0528                               "/home/oracle/setup/…"   2 days ago          Up 2 days                               k8s_database_database-67d9b8bd67-hnrtl_default_ef55e8e6-5881-42a7-a1dc-1334d3534b9e_0
85d440786ff5        k8s.gcr.io/pause:3.2                       "/pause"                 2 days ago          Up 2 days                               k8s_POD_database-67d9b8bd67-hnrtl_default_ef55e8e6-5881-42a7-a1dc-1334d3534b9e_0
883fe2240f8a        quay.io/k8scsi/csi-resizer                 "/csi-resizer --csi-…"   3 days ago          Up 3 days                               k8s_csi-resizer_pure-provisioner-0_pso_e9016f25-a628-49ed-b914-51a056ac9cd5_0
016844ac1651        quay.io/k8scsi/csi-snapshotter             "/csi-snapshotter --…"   3 days ago          Up 3 days                               k8s_csi-snapshotter_pure-provisioner-0_pso_e9016f25-a628-49ed-b914-51a056ac9cd5_0
fba49a591a75        quay.io/k8scsi/csi-provisioner             "/csi-provisioner --…"   3 days ago          Up 3 days                               k8s_csi-provisioner_pure-provisioner-0_pso_e9016f25-a628-49ed-b914-51a056ac9cd5_0
06eaefb14fb1        purestorage/k8s                            "/csi-server -endpoi…"   3 days ago          Up 3 days                               k8s_pure-csi-container_pure-provisioner-0_pso_e9016f25-a628-49ed-b914-51a056ac9cd5_0
2f89ff9880e9        k8s.gcr.io/pause:3.2                       "/pause"                 3 days ago          Up 3 days                               k8s_POD_pure-provisioner-0_pso_e9016f25-a628-49ed-b914-51a056ac9cd5_0

```

- log into the oracle container and check the persistent volume is mounted and Oracle has started
```
# docker exec -it f4c2a4904845 bash
[oracle@database-67d9b8bd67-hnrtl /]$
[oracle@database-67d9b8bd67-hnrtl /]$
[oracle@database-67d9b8bd67-hnrtl /]$ df -k
Filesystem                                    1K-blocks     Used Available Use% Mounted on
overlay                                        47797996 17521468  27818776  39% /
tmpfs                                             65536        0     65536   0% /dev
tmpfs                                           4084048        0   4084048   0% /sys/fs/cgroup
/dev/mapper/3624a9370a21265762db64ece00054b1b  20961280  4480188  16481092  22% /ORCL
/dev/sda5                                      47797996 17521468  27818776  39% /etc/hosts
tmpfs                                           4084048        0   4084048   0% /dev/shm
tmpfs                                           4084048       12   4084036   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                           4084048        0   4084048   0% /proc/acpi
tmpfs                                           4084048        0   4084048   0% /proc/scsi
tmpfs                                           4084048        0   4084048   0% /sys/firmware
```

- Confirm the persistenet volume is being used
Exit the container and log back into the worker node
```
run the lsblk command
sdh                                   8:112  0    20G  0 disk
└─3624a9370a21265762db64ece00054b1b 253:1    0    20G  0 mpath /var/lib/kubelet/pods/ef55e8e6-5881-42a7-a1dc-1334d3534b9e/volumes/kubernetes.io~csi/pvc-5601093c-b828-4255-8a85-5fe1aed2d166/mount

we can see the /dev/mapper volume 00054b1b from the container has been mapped to the PV volume created earlier
# kubectl get pv|grep oravol
pvc-5601093c-b828-4255-8a85-5fe1aed2d166   20Gi       RWO            Delete           Bound    default/oravol    pure-block              3d15h
```

- Log back into the Container and confirm Oracle has stated and we can access the database
```
#$ ps -ef|grep pmon
oracle      44     1  0 May08 ?        00:00:15 ora_pmon_ORCL
oracle   11532 11469  0 00:41 pts/1    00:00:00 grep --color=auto pmon
[oracle@database-67d9b8bd67-hnrtl /]$

# cd $ORACLE_HOME

[oracle@database-67d9b8bd67-hnrtl dbhome_1]$ sqlplus / as sysdba
SQL*Plus: Release 12.2.0.1.0 Production on Mon May 11 00:42:18 2020
Copyright (c) 1982, 2016, Oracle.  All rights reserved.

Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select instance_name, host_name from v$instance;

INSTANCE_NAME
----------------
HOST_NAME
----------------------------------------------------------------
ORCL
database-67d9b8bd67-hnrtl
SQL>

```

- Stop the database
```
kubectl scale -n ora deployment database --replicas=0
```
- start the instance
```
kubectl scale -n ora deployment database --replicas=1
```

# Oracle yaml file

NOTE: The securityContext is the Pod Security Context, it supports setting an fsGroup, which allows you to set the group ID that owns the volume, and thus who can write to it.  If you dont add this you will get permission errors when trying to write to separate volume for logs
```
  securityContext:
        fsGroup: 54321
```     
        
* refer to the oracle.yaml file on my github
https://github.com/bannaych/kubernetes/blob/master/oracle.yaml
