# create k8s by kubeadm

**_root needed_**

## ubuntu

```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
#close swap
swapoff -a
#remove swap mount
vi /etc/fstab
```

> apt-key is deprecated in some new version linux distribution, like ubuntu, put gpg file inside /etc/apt/trusted.gpg.d

## fedora

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#close selinux
setenforce 0
#disable selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g'
#disable firewalld
systemctl disable firewalld.service

dnf update && dnf install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
#close swap
swapoff -a
#disable swap
dnf remove zram-generator-defaults
```

## config

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
  - groups:
      - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
      - signing
      - authentication
kind: InitConfiguration
localAPIEndpoint:
  #apiserver address
  advertiseAddress: 192.168.1.2
  bindPort: 6443
nodeRegistration:
  # use containerd, docker is deprecated
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  # node must visis
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
dns: {}
etcd:
  external:
    # external etcd, you may use etcd provide by kubeadm
    endpoints:
      - https://192.168.1.2:2379
    caFile: /etc/etcd/pki/ca.crt
    certFile: /etc/etcd/pki/apiserver-etcd-client.crt
    keyFile: /etc/etcd/pki/apiserver-etcd-client.key

kind: ClusterConfiguration
# latest version
kubernetesVersion: 1.24.1
networking:
  dnsDomain: cluster.local
  # here we use flannel cni plugin, you may change you use others such as calico
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

> you can use `kubeadm config print init-defaults` print the latest version

## change kernel param

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
EOF

```

maybe need load immediately

```bash
modprobe br_netfilter overlay
```

```bash
#pull image first
kubeadm config images pull --config kubeadm-init-config.yaml
#init
kubeadm init --config kubeadm-init-config.yaml
#allow single node cluster run pod
kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
```

```bash
mkdir $HOME/.kube && sudo cp /etc/kubenertes/admin.conf $HOME/.kube/config
kubectl get node
```

you will see k8s pod runing
