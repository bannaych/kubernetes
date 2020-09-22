
# SQL Server containers on Kubernetes and Pure Storage with Metallb LoadBalencer

In this Blog I will show you how we can use a combination of SQL Server, Pure Service Orchestrator ( PSO ) and MetalLB to create a load balanced Kubernetes clyuster
for SQL Server

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.


# Environment

|Role|FQDN|IP|OS|RAM|CPU|K8 Ver
|----|----|----|----|----|----|---|
|Master|k8master|192.168.111.228|Ubuntu 18.04|8G|3|1.18.2
|Worker1|k8worker1|192.168.111.229|Ubuntu 18.047|8G|4|1.18.2
|Worker2|k8worker2|192.168.111.230|Ubuntu 18.047|8G|4|1.18.2

I already have an Ubuntu Cluster up and running so i wont be going through the install process in this blog

- Lets check the cluster status

```
root@k8master:~# kubectl get nodes
NAME        STATUS   ROLES    AGE    VERSION
k8master    Ready    master   140d   v1.18.2
k8worker1   Ready    <none>   140d   v1.18.2
k8worker2   Ready    <none>   140d   v1.18.2
```

- Lets check to see if we have any pod or services deployed

```
root@k8master:~# kubectl get pods
No resources found in default namespace.

oot@k8master:~# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hollowdb     ClusterIP   10.111.175.138   <none>        3306/TCP   140d
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    140d
```

As you can see from the above output, we have no PODS and no SERVICES.

- Now lets create a new SQL Server deployment called mssql-deployment

```
root@k8master:~# kubectl apply -f cbsql.yaml
persistentvolumeclaim/pvc-sql-system unchanged
deployment.apps/mssql-deployment configured
service/mssql-deployment created
```

- Let check to make sure out POD is running

```
root@k8master:~# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
mssql-deployment-df565f596-zkb87   1/1     Running   0          48s
```

- Let have a closer look, as you can see, the POD has started on node K8worker2, its up and running and use the 2019-CU4-ubuntu-18.04 image

```
root@k8master:~# kubectl describe pods mssql-deployment
Name:         mssql-deployment-df565f596-zkb87
Namespace:    default
Priority:     0
Node:         k8worker2/192.168.111.230
Start Time:   Tue, 22 Sep 2020 10:03:37 +1000
Labels:       app=mssql
              pod-template-hash=df565f596
Annotations:  <none>
Status:       Running
IP:           10.244.2.18
IPs:
  IP:           10.244.2.18
Controlled By:  ReplicaSet/mssql-deployment-df565f596
Containers:
  mssql:
    Container ID:   docker://db2f3fcfc04b0717db90a146435a2f5476bc88bd415a67710a123ecbe7e1e80e
    Image:          mcr.microsoft.com/mssql/server:2019-CU4-ubuntu-18.04
    Image ID:       docker-pullable://mcr.microsoft.com/mssql/server@sha256:360f6e6da94fa0c5ec9cbe6e391f411b8d6e26826fe57a39a70a2e9f745afd82
    Port:           1433/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 22 Sep 2020 10:03:43 +1000
    Ready:          True
    Restart Count:  0
    Environment:
      ACCEPT_EULA:  Y
      SA_PASSWORD:  <set to the key 'SA_PASSWORD' in secret 'mssql'>  Optional: false
    Mounts:
      /var/opt/mssql from mssql-system (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-kj8td (ro)

```

At this point I can connect to the database using the post specified the service yaml code ( 31433 ), It's a cluser wide port number however, I need to specify
the worker node IP address, so if the pod moves to another I need to specify the other nodes IP address. A better way is to create a load-Balanced IP address which allows me to connec to one IP address regardless of which node the service is running on. This is where MetalLB comes into play.
```
---
apiVersion: v1
kind: Service
metadata:
  name: mssql-deployment
spec:
  selector:
    app: mssql
  ports:
    - protocol: TCP
      port: 31433
      targetPort: 1433
  type: LoadBalancer
 ```
 
 # Lets install and configure MetalLB.
 
 - download metalLB from https://metallb.universe.tf/
 
 - select the installation TAB, select installation manifest, copy the code and run it on your cluster
 ```
 kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

```

Ok lets check to make sure metalLB is install, first lets check to make sure the namespace is created, as you call from the below output, we can see
the metallb-system namespace has been created.

```
root@k8master:~# kubectl get ns
NAME              STATUS   AGE
default           Active   140d
metallb-system    Active   12h
oradb             Active   139d
pso               Active   140d
```

Next we need to deloy the configuration, we will deploy this a configmap. Go to the configuration tab on the metallb web page
select the Layer 2 configuration
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.111.240-192.168.111.245
```

- In the above configmap, I have confgured my Loadbalancer to use the addresses from 192.168.111.240 - 192.168.111.245, this means when the service starts
it will grab an IP from that range and assign it to the LoadBalancer service.

- So Lets create the configmap and have a look at it, as you can see, the configmap called config has been created with the address range I specified

```
root@k8master:~# kukectl create -f configmap
configmap/config created

root@k8master:~# kubectl describe configmap config -n metallb-system
Name:         config
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>

Data
====
config:
----
address-pools:
- name: default
  protocol: layer2
  addresses:
  - 192.168.111.240-192.168.111.245
```




