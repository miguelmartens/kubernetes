# Kubernetes provisioning with kubeadm and CRI-O

Follow this documentation to provision a single node Kubernetes cluster with kubeadm on __Ubuntu 20.04 LTS__ with __cri-o__ as the container runtime.


```
sudo su -

apt update && apt upgrade -y
```


##### Disable swap and firewall
```
{

sed -i '/swap/d' /etc/fstab
swapoff -a
systemctl disable --now ufw

}
```

##### Update /etc/hosts
```
cat >>/etc/hosts<<EOF
192.168.50.100   k8s.cryocore.io     cryocore
EOF
```

##### Kernel modules and sysctl settings
```
{

cat >>/etc/modules-load.d/crio.conf<<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

}
```

##### Install cri-o container runtime
```
{

OS=xUbuntu_20.04
VERSION=1.21

cat >>/etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list<<EOF
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF

cat >>/etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list<<EOF
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers-cri-o.gpg add -

apt update && apt install -qq -y cri-o cri-o-runc cri-tools

cat >>/etc/crio/crio.conf.d/02-cgroup-manager.conf<<EOF
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "systemd"
EOF

systemctl daemon-reload
systemctl enable --now crio

}
```

##### Install Kubernetes components
```
{
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
apt install -qq -y kubeadm=1.21.0-00 kubelet=1.21.0-00 kubectl=1.21.0-00
}
```

##### Initialize kubernetes cluster
```
kubeadm init --apiserver-advertise-address=192.168.50.100 --pod-network-cidr=192.168.0.0/16
```

##### Timeout issue
```
# more info at https://github.com/cri-o/cri-o/blob/master/tutorials/kubeadm.md
KUBELET_EXTRA_ARGS="--feature-gates=\"AllAlpha=false,RunAsGroup=true\" --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock' --runtime-request-timeout=5m"
# add Kublet extra args to /etc/default/kubelet if not there
grep -qxF "KUBELET_EXTRA_ARGS=${KUBELET_EXTRA_ARGS}" /etc/default/kubelet \
|| echo "KUBELET_EXTRA_ARGS=${KUBELET_EXTRA_ARGS}" \
| sudo tee /etc/default/kubelet
```

##### Deploy network add on - Calico
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Downloading kube config to your local machine 
On your host machine
```
mkdir ~/.kube
scp root@192.168.50.100:/etc/kubernetes/admin.conf ~/.kube/config
```

### Verifying the cluster
```
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

### Scheduling Pods on the control-plane node
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

```
wget https://raw.githubusercontent.com/miguelmartens/kubernetes/main/various/kubernetes-crio/crio.conf -P /etc/crio/crio.conf
systemctl restart crio
crictl ps
```
