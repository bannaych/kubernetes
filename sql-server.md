
# SQL Server containers on Kubernetes and Pure Storage with Metallb LoadBalencer

In this Blog I will show you how we can use a combination of SQL Server, Pure Service Orchestrator ( PSO ) and MetalLB to create a load balanced Kubernetes clyuster
for SQL Server

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.


# Environment

|Role|FQDN|IP|OS|RAM|CPU|K8 Ver
|----|----|----|----|----|----|---|
|Master|k8master|192.168.111.228|Ubuntu 18.04|8G|3|1.18
|Worker1|k8worker1|192.168.111.229|Ubuntu 18.047|8G|4|1.18
|Worker2|k8worker2|192.168.111.230|Ubuntu 18.047|8G|4|1.18

I already have an Ubuntu Cluster up and running so i wont be going through the install process in this blog

- Lets check the cluster status

```
root@k8master:~# kubectl get nodes
NAME        STATUS   ROLES    AGE    VERSION
k8master    Ready    master   140d   v1.18.2
k8worker1   Ready    <none>   140d   v1.18.2
k8worker2   Ready    <none>   140d   v1.18.2
```
