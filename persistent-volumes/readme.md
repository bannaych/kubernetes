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

* Create the Persistent volume
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
