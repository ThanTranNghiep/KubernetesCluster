# Install Kubernetes Cluster using kubeadm
Follow this documentation to set up a Kubernetes cluster on __ORACLE LINUX 8.9__.

This documentation guides you in setting up a cluster with one master node and two worker node.

## Assumptions
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|master|10.0.0.20|Oracle Linux 8.9|7.5G|4|
|Worker|worker01|10.0.0.21|Oracle Linux 8.9|7.5G|4|
|Worker|worker02|10.0.0.22|Oracle Linux 8.9|7.5G|4|

## On both Kmaster and Kworker
Perform all the commands as root user unless otherwise specified
##### Disable Firewall
```
systemctl disable firewalld; systemctl stop firewalld
```
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Disable SELinux
```
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1
EOF
sysctl --system
```
##### Install docker engine
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce
systemctl enable --now docker
```
##### Config container
```
cat > /etc/containerd/config.toml <<EOF
[plugins."io.containerd.grpc.v1.cri"]
  systemd_cgroup = true
EOF
systemctl restart containerd
```
### Kubernetes Setup
##### Add yum repository
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
##### Install Kubernetes components
```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
##### Enable and Start kubelet service
```
systemctl enable --now kubelet
```
## On kmaster
##### Initialize Kubernetes Cluster
```
kubeadm init --apiserver-advertise-address=10.0.0.20 --pod-network-cidr=192.168.0.0/16
```
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml
```
##### Cluster join command
```
kubeadm token create --print-join-command
```
##### To be able to run kubectl commands as non-root user
If you want to be able to run kubectl commands as non-root user, then as a non-root user perform these
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## On Kworker
##### Config container
```
cat > /etc/containerd/config.toml <<EOF
[plugins."io.containerd.grpc.v1.cri"]
  systemd_cgroup = true
EOF
systemctl restart containerd
```
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Verifying the cluster
##### Get Nodes status
```
kubectl get nodes
```
##### Get component status
```
kubectl get cs
```

### After adding worker node to master
- On master
```cmd
[root@master ~]# 
[root@master ~]# kubectl get po -n kube-system -w
NAME                                      READY   STATUS    RESTARTS       AGE
calico-kube-controllers-6dc98bdbf-4pzdv   1/1     Running   13 (99m ago)   2d15h
calico-node-j98zw                         1/1     Running   3 (99m ago)    2d15h
calico-node-kkbd8                         1/1     Running   9 (100m ago)   2d16h
calico-node-xfkdt                         1/1     Running   0              90m
coredns-7db6d8ff4d-s2fkt                  1/1     Running   3 (100m ago)   2d16h
coredns-7db6d8ff4d-zg8hz                  1/1     Running   3 (100m ago)   2d16h
etcd-master                               1/1     Running   3 (100m ago)   2d16h
kube-apiserver-master                     1/1     Running   3 (100m ago)   2d16h
kube-controller-manager-master            1/1     Running   4 (99m ago)    2d16h
kube-proxy-bf244                          1/1     Running   3 (100m ago)   2d16h
kube-proxy-lhrss                          1/1     Running   0              90m
kube-proxy-ltg72                          1/1     Running   3 (99m ago)    2d15h
kube-scheduler-master                     1/1     Running   4 (99m ago)    2d16h
```
OR
```cmd
[root@master ~]# kubectl get po -A -o wide
NAMESPACE     NAME                                      READY   STATUS    RESTARTS        AGE     IP               NODE       NOMINATED NODE   READINESS GATES
default       nginx-deployment-5449cb55b-cfkrr          1/1     Running   3 (102m ago)    2d15h   192.168.5.23     worker01   <none>           <none>
default       nginx-deployment-5449cb55b-mb42z          1/1     Running   3 (102m ago)    2d15h   192.168.5.22     worker01   <none>           <none>
default       nginx-deployment-5449cb55b-ttpdz          1/1     Running   3 (102m ago)    2d15h   192.168.5.24     worker01   <none>           <none>
kube-system   calico-kube-controllers-6dc98bdbf-4pzdv   1/1     Running   13 (101m ago)   2d15h   192.168.5.21     worker01   <none>           <none>
kube-system   calico-node-j98zw                         1/1     Running   3 (102m ago)    2d15h   10.0.0.21        worker01   <none>           <none>
kube-system   calico-node-kkbd8                         1/1     Running   9 (103m ago)    2d16h   10.0.0.20        master     <none>           <none>
kube-system   calico-node-xfkdt                         1/1     Running   0               92m     10.0.0.22        worker02   <none>           <none>
kube-system   coredns-7db6d8ff4d-s2fkt                  1/1     Running   3 (103m ago)    2d16h   192.168.219.72   master     <none>           <none>
kube-system   coredns-7db6d8ff4d-zg8hz                  1/1     Running   3 (103m ago)    2d16h   192.168.219.71   master     <none>           <none>
kube-system   etcd-master                               1/1     Running   3 (103m ago)    2d16h   10.0.0.20        master     <none>           <none>
kube-system   kube-apiserver-master                     1/1     Running   3 (103m ago)    2d16h   10.0.0.20        master     <none>           <none>
kube-system   kube-controller-manager-master            1/1     Running   4 (101m ago)    2d16h   10.0.0.20        master     <none>           <none>
kube-system   kube-proxy-bf244                          1/1     Running   3 (103m ago)    2d16h   10.0.0.20        master     <none>           <none>
kube-system   kube-proxy-lhrss                          1/1     Running   0               92m     10.0.0.22        worker02   <none>           <none>
kube-system   kube-proxy-ltg72                          1/1     Running   3 (102m ago)    2d15h   10.0.0.21        worker01   <none>           <none>
kube-system   kube-scheduler-master                     1/1     Running   4 (101m ago)    2d16h   10.0.0.20        master     <none>           <none>
[root@master ~]# 
```
***
## FIX issue on worker

1. Couldn't get current server API group list
 ```cmd
 [root@worker01 ~]# kubectl get nodes
E0812 09:59:43.682577   19865 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0812 09:59:43.683911   19865 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0812 09:59:43.685248   19865 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0812 09:59:43.686413   19865 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E0812 09:59:43.687448   19865 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
[root@worker01 ~]# 
```

**Solution:**
- On master
```cmd
scp /etc/kubernetes/admin.conf root@worker02:/tmp/admin.conf
```
- On worker
```
mkdir -p $HOME/.kube
cp -i /tmp/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Result on worker
```cmd
[root@worker01 ~]# kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
master     Ready    control-plane   2d16h   v1.30.3
worker01   Ready    <none>          2d15h   v1.30.3
worker02   Ready    <none>          88m     v1.30.3
[root@worker01 ~]# 
```
