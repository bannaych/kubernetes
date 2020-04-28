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
