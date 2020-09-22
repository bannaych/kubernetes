
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

- Let have a closer look at that POD

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
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  mssql-system:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-sql-system
    ReadOnly:   false
  default-token-kj8td:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-kj8td
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                Message
  ----    ------     ----       ----                -------
  Normal  Scheduled  <unknown>  default-scheduler   Successfully assigned default/mssql-deployment-df565f596-zkb87 to k8worker2
  Normal  Pulling    4m23s      kubelet, k8worker2  Pulling image "mcr.microsoft.com/mssql/server:2019-CU4-ubuntu-18.04"
  Normal  Pulled     4m22s      kubelet, k8worker2  Successfully pulled image "mcr.microsoft.com/mssql/server:2019-CU4-ubuntu-18.04"
  Normal  Created    4m22s      kubelet, k8worker2  Created container mssql
  # Normal  Started    4m21s      kubelet, k8worker2  Started container mssql
```

