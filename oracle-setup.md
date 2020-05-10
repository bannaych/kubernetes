# Oracle Containers on Kubernetes and Pure Storage PSO

PSO allow for smart provisioning of storage to one or many arrays, this provides administrator and developers the ability to  provision storage on demand with cloud like functionality, ease and flexibility.

In this blog I will use Pure Storage Orchestrator ( PSO ) to automate the provisioning of volumes to Kubernetes which will then be used to create Oracle containers using persistent storage.  

My next blog will demonstrate the  power or Pure Storage snapshots and cloning to rapidly create instant copies of the source Oracle database.

# Environment

|Role|FQDN|IP|OS|RAM|CPU|K8 Ver
|----|----|----|----|----|----|---|
|Master|master.localdomain.com|192.168.111.231|Ubuntu 18.04|5G|3|18.1
|Worker1|worker1.localdomain.com|192.168.111.232|Ubuntu 18.047|8G|4|18.1

# Install PSO

- Install helm 3
```
# curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
# ./get_helm.sh

```


In this example, we will use an separate Oracle namespace to configure Oracle
this is not mandatory, however it easier to manage when you have multiple application and pods running 
on the same system


* Create the persistent volumes for Oracle, is this example I will create a data volume and log volume
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: oravol
  namespace: ora
  labels:
    app: oracle12c
spec:
 accessModes:
 - ReadWriteOnce
 resources:
  requests:
   storage: 20Gi
 storageClassName: pure-block
 ```
 ```
 # kubectl create -f oravol.yaml
 persistentvolumeclaim/oravol created

 
kubectl get pvc -n ora
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
oralog   Bound    pvc-89bdc6ad-a5ba-4041-bb9c-09d0b73900c7   20Gi       RWO            pure-block     27m

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-89bdc6ad-a5ba-4041-bb9c-09d0b73900c7   20Gi       RWO            Delete           Bound    ora/oralog        pure-block              28m
 ```
 
* Create the Oracle namespace
```
# kubectl create namespace -n ora
# kubectl get namespace ora
NAME   STATUS   AGE
ora    Active   3h13m
```

* Create Kubernets secret to include the docker login information
```
kubectl create secret docker-registry oracle --docker-server=docker.io --docker-username=bannaych --docker-password=PASSWORD --docker-email=bannaych@gmail.com -n ora
```

In this example I am not using a configmap file. I have includes the oracle variables inside the main yaml file, however you can create
a properties file if you prefer.

* Crete and Start the database
```
kubectl apply -f oracle.yaml
deployment.apps/database configured
```
```
get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-hj6gp         1/1     Running   0          17h
kube-system   coredns-66bff467f8-sjqqg         1/1     Running   0          17h
kube-system   etcd-master                      1/1     Running   0          17h
kube-system   kube-apiserver-master            1/1     Running   0          17h
kube-system   kube-controller-manager-master   1/1     Running   0          17h
kube-system   kube-flannel-ds-amd64-xtjg8      1/1     Running   0          17h
kube-system   kube-flannel-ds-amd64-zppzb      1/1     Running   0          17h
kube-system   kube-proxy-jtv9v                 1/1     Running   0          17h
kube-system   kube-proxy-ztq2w                 1/1     Running   0          17h
kube-system   kube-scheduler-master            1/1     Running   0          17h
ora           database-7544cd4df7-d4hm4        1/1     Running   0          6s
pso           pure-csi-mgl76                   3/3     Running   0          17h
pso           pure-provisioner-0               4/4     Running   0          17h
```

* Check the status of the Oracle build
```
# kubectl logs database-7544cd4df7-d4hm4 -n ora

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
Listener Log File         /u01/app/oracle/diag/tnslsnr/database-7544cd4df7-d4hm4/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
The listener supports no services
The command completed successfully

DONE!
```

* Log onto the worker node

```
# docker ps|grep ora
e34d786e6a6b        12a359cd0528                               "/home/oracle/setup/…"   31 minutes ago      Up 31 minutes                           k8s_database_database-7544cd4df7-d4hm4_ora_834ddc41-6416-4685-9961-b57903fa1dec_0
7fd7682977dc        k8s.gcr.io/pause:3.2                       "/pause"                 31 minutes ago      Up 31 minutes                           k8s_POD_database-7544cd4df7-d4hm4_ora_834ddc41-6416-4685-9961-b57903fa1dec_0
b194d4dc3024        purestorage/k8s                            "/csi-server -endpoi…"   18 hours ago        Up 18 hours                             k8s_pure-csi-container_pure-csi-mgl76_pso_ea88ab80-c662-4769-9f1e-8a099218af07_0
f351b6f9fe1b        purestorage/k8s                            "/csi-server -endpoi…"   18 hours ago        Up 18 hours                             k8s_pure-csi-container_pure-provisioner-0_pso_844495f7-42f9-43f4-b70b-28c8ce9749a2_0
```

* log into the oracle container and check the persistent volume is mounted and Oracle has started
```
# docker exec -it e34d786e6a6b bash
[oracle@database-7544cd4df7-d4hm4 /]$
[oracle@database-7544cd4df7-d4hm4 /]$
[oracle@database-7544cd4df7-d4hm4 /]$ df -k
Filesystem                                    1K-blocks     Used Available Use% Mounted on
overlay                                        47797996 17731868  27608376  40% /
tmpfs                                             65536        0     65536   0% /dev
tmpfs                                           4084204        0   4084204   0% /sys/fs/cgroup
/dev/mapper/3624a9370a21265762db64ece000547bf  20961280  3990716  16970564  20% /ORCL
tmpfs                                           4084204        0   4084204   0% /dev/shm
/dev/sda5                                      47797996 17731868  27608376  40% /etc/hosts
tmpfs                                           4084204       12   4084192   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                           4084204        0   4084204   0% /proc/acpi
tmpfs                                           4084204        0   4084204   0% /proc/scsi
tmpfs                                           4084204        0   4084204   0% /sys/firmware

#ps -ef|grep pmon
oracle     174     1  0 01:30 ?        00:00:00 ora_pmon_ORCL
oracle     946   921  0 02:02 pts/0    00:00:00 grep --color=auto pmon

# cd $ORACLE_HOME

[oracle@database-7544cd4df7-d4hm4 dbhome_1]$ sqlplus / as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Thu May 7 02:03:47 2020

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select instance_name, host_name from v$instance;

INSTANCE_NAME
----------------
HOST_NAME
----------------------------------------------------------------
ORCL
database-7544cd4df7-d4hm4

```

* Stop the database
```
kubectl scale -n ora deployment database --replicas=0
```
* start the instance
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
