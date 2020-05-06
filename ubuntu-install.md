# Installing Kubernetes on Ubuntu

```
# apt-get install docker.io -y
```

```
# systemctl start docker
# systemctl start docker
```

```
# apt-get install apt-transport-https curl -y
```
```
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```
```
# apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```
```
# apt update
```

turn off Swap
```
# swapoff -a
```
```
# apt-get install kubeadm -y
```
```
# apt-get install -y kubelet kubectl kubernetes-cni
```
```
# kubeadm init --pod-network-cidr=10.244.0.0/16
```
```
# mkdir -p $HOME/.kube
# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
```
```
# Kubectl get nodes
```
```
# sysctl net.bridge.bridge-nf-call-iptables=1
```
```
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
```
# kubectl get pods --all-namespaces
# kubectl get nodes
```

