
# SQL Server containers on Kubernetes and Pure Storage with Metallb LoadBalencer

In this Blog I will show you how we can use a combination of SQL Server, Pure Service Orchestrator ( PSO ) and MetalLB to create a load balanced Kubernetes clyuster
for SQL Server

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.


# Environment

|Role|FQDN|IP|OS|RAM|CPU|K8 Ver
|----|----|----|----|----|----|---|
|Master|k8master|192.168.111.228|Ubuntu 18.04|5G|8|1.18
|Worker1|k8worker1|192.168.111.229|Ubuntu 18.047|8G|4|1.18
|Worker2|k8worker2|192.168.111.230|Ubuntu 18.047|8G|4|1.18

