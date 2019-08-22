                                         
# kubernetes


### Kubernets官方提供三种部署方式：

#### minikube
minikube可以实现一种轻量级的Kubernetes集群，通过在本地计算机上创建虚拟机并部署只包含单个节点的简单集群。Minikube适用于Linux，MacOS和Windows系统。Minikube CLI提供集群管理的基本操作，包括 start、stop、status和delete

#### kubeadm
kubeadm是Kubernetes1.6开始官方推出的快速部署Kubernetes集群工具，其思路是将Kubernetes相关服务容器化(Kubernetes静态Pod)以简化部署

#### custom solutions
最完整的方式，从零开始二进制搭建  

<br>

## 部署步骤

### 准备环境
```
# 关闭防火墙：
# systemctl stop firewalld
# systemctl disable firewalld

# 关闭selinux：
# sed -i 's/enforcing/disabled/' /etc/selinux/config 
# setenforce 0

# 关闭swap：
# swapoff -a  # 临时
# vim /etc/fstab  # 永久

# vim /etc/sysctl.conf:
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1


# 设置主机名：
# hostnamectl set-hostname master
  
# 修改/etc/hosts文件:
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.105    k8s-1
192.168.0.106    k8s-2
192.168.0.107    k8s-3

# 修改网络配置成静态ip,然后:
# systemctl restart network
```

### 安装docker
```
# yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum list docker-ce --showduplicates | sort -r
# yum install docker-ce-18.03.1.ce-1.el7.centos
# systemctl start docker
# systemctl enable docker
```

### 安装k8s仓库
```
# cd /etc/yum.repos.d/
# cat>>kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### 安装kubectl
```
# yum install kubectl
```

### 使用minikube安装k8s
```
# 安装minikube
# curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.35.0/minikube-linux-amd64
# chmod +x minikube 
# mv minikube /usr/bin/minikube

# minikube start --registry-mirror=https://registry.docker-cn.com --kubernetes-version v1.12.0 --vm-driver=none
```

### 使用kubeadm安装k8s
```
# 安装kubelet
# yum install kubeadm kubelet
# 修改kubelet启动配置文件，主要是将--cgroup-driver改为cgroupfs（确保和/usr/lib/systemd/system/docker.service的用户一致就可以了，不需要修改!)
# cat  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig=true"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_EXTRA_ARGS

# systemctl start kubelet
# systemctl enable kubelet
  
# 提前下载镜像，国内因为gcr.io被墙，所以要么通过代理翻墙获取，要么寻找其他办法。可以访问
# https://hub.docker.com/,然后搜索kube-apiserver-amd64,会列出已经build好的images，
# 选择相应的版本，进行pull
  
#也可以从aliyun上进行下载：

# K8S_VERSION=v1.15.3
# ETCD_VERSION=3.3.10
# DASHBOARD_VERSION=v1.8.3
# FLANNEL_VERSION=v0.10.0-amd64
# DNS_VERSION=1.3.1
# PAUSE_VERSION=3.1
# 基本组件
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$DNS_VERSION
# 网络组件
# docker pull quay.io/coreos/flannel:$FLANNEL_VERSION
# 修改tag
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION k8s.gcr.io/kube-apiserver:$K8S_VERSION
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION k8s.gcr.io/kube-controller-manager:$K8S_VERSION
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION k8s.gcr.io/kube-scheduler:$K8S_VERSION
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION k8s.gcr.io/kube-proxy:$K8S_VERSION
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$DNS_VERSION k8s.gcr.io/coredns:$DNS_VERSION

# kubeadm init --pod-network-cidr=10.244.0.0/16
# 备用：kubeadm init --kubernetes-version=v1.7.2 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=0.0.0.0 --apiserver-cert-extra-sans=192.168.0.105,192.168.0.106,192.168.0.107,127.0.0.1,k8s-1,k8s-2,k8s-3,192.168.0.1 --skip-preflight-checks 
  
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml

# kubectl taint nodes --all node-role.kubernetes.io/master-
```
  
### 安装helm
```
# wget https://get.helm.sh/helm-v2.14.3-linux-amd64.tar.gz
# tar -zxvf helm-v2.14.3-linux-amd64.tar.gz
# cd linux-amd64
# mv helm /usr/bin
# helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

### helm命令补全
```
# source <(helm completion bash)
```
  
### 安装dashboard
```
# kubectl apply -f http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
# kubectl -n kube-system edit service kubernetes-dashboard  #type: NodePort
# cat>>dashboard-admin.yaml<<EOF
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels: 
     k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
EOF
# kubectl apply -f dashboard-admin.yaml
```