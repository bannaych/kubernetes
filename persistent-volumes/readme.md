# Peristent Volumes in Kubernetes

In this example we will create a persistent Volume, followed by a persistent volume
claim, then create an nginx pod that will mount the volume we created

As this is my home environment, I have an Ubuntu server sharing and NFS filesystm over running on ZFS

Check for PV
```
kubectl get pv
No resources found in default namespace. 
```
Check for PVC
```
kubectl get pvc
No resources found in default namespace.
```

* Create the yaml file
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs1
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.0.240
    path: "/mypool/data1"
```

* Create the PV
```
kubectl create -f nfspv.yaml
persistentvolume/pv-nfs1 created

kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-nfs1   5Gi        RWX            Retain           Available           manual                  14s
```

* Create the PVC yaml file
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-nfs1
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

* Create the PVC
```
kubectl create -f nfs-pvc.yaml
persistentvolumeclaim/pv-nfs1 created

kubectl get pvc
NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pv-nfs1   Bound    pv-nfs1   5Gi        RWX            manual         14s
```

* Create the nginx yaml file, note the pvc volume **pv-nfs1**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: mynginx
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: pv-nfs1
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
```
* Create the nginx pod

```
kubectl create -f deploynfs.yaml
deployment.apps/mynginx created

kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
mynginx   1/1     1            1           27s

```
* Get the name of the new POD
```
kubectl get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/mynginx-58854674df-vnrgp   1/1     Running   0          2m20s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          4d1h
service/mynginx      NodePort    10.111.183.18   <none>        80:32443/TCP     4d
service/oracle12c    NodePort    10.109.93.8     <none>        1521:30759/TCP   24h

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mynginx   1/1     1            1           2m20s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/mynginx-58854674df   1         1         1       2m20s
```

* Fine out which workder node the container is running on ( worker2 )
```
kubectl describe pod mynginx-58854674df-vnrgp
Name:         mynginx-58854674df-vnrgp
Namespace:    default
Priority:     0
Node:         worker2.localdomain/192.168.0.232
Start Time:   Tue, 28 Apr 2020 21:22:26 +1000
Labels:       pod-template-hash=58854674df
              run=nginx
              
```

* SSH into worker 2 and check the filesystem is mounted
```
mount|grep data
192.168.0.240:/mypool/data1 on /var/lib/kubelet/pods/10906cfd-ff44-4970-83d0-7c558d88acd4/volumes/kubernetes.io~nfs/pv-nfs1
```
* Log into the Container and confirm nginx is running on the Persistent volume
```
docker exec -it f3122341bb94 bash
root@mynginx-58854674df-vnrgp:/# df -k
Filesystem                   1K-blocks    Used  Available Use% Mounted on
overlay                       37722116 7164592   30557524  19% /
tmpfs                            65536       0      65536   0% /dev
tmpfs                          4987368       0    4987368   0% /sys/fs/cgroup
/dev/mapper/ol-root           37722116 7164592   30557524  19% /etc/hosts
shm                              65536       0      65536   0% /dev/shm
192.168.0.240:/mypool/data1 1450573824       0 1450573824   0% /usr/share/nginx/html
tmpfs                          4987368      12    4987356   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                          4987368       0    4987368   0% /proc/acpi
tmpfs                          4987368       0    4987368   0% /proc/scsi
tmpfs                          4987368       0    4987368   0% /sys/firmware
```
