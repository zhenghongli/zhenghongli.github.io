---
title: Installing Kubernets CRI-O in RHEL 8.4
date: 2023-02-23
publishdate: 2023-04-23
categories:
- Kubernetes
tags:
- CRI-O
- Kubernetes
keywords:
- tech
comments:       false
showMeta:       false
showActions:    false
---

K8S(Kubernetes)近年來發展的十分迅速，Red Hat也出了十分知名的OCP(Openshift Container Platform)，而其中OCP就是選擇CRI-O作為K8S的CRI(Container Runtime)。由於本篇幅主要是介紹使用，在此僅稍微介紹CRI-O。
<!--more-->

CRIO-O是CNCF所孵化的開源專案，貢獻者包含Red Hat、IBM、 Intel、SUSE等大廠，他是相較於Docker較為輕量化的CRI，主要目的在於讓K8S控制Container，因此他不具有Build Container以及建立虛擬網路的功能，可以說是專門為了K8S而生，而非像Docker是注重在Container的操作。




## 架設環境
此環境僅作為設置參考，請根據需求酌量調整

Master * 1
* OS: RHEL8.4
* CPU: 4 core
* Memory: 8G
* Disk: 50GB

Worker * 2
* OS: RHEL8.4
* CPU: 4 core
* Memory: 8G
* Disk: 50GB

## 所有節點安裝步驟

### Installing CRI-O
```
export OS=CentOS_8
export VERSION=1.22
```

```
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
```

`sudo yum install cri-o`

```
systemctl enable crio.service
systemctl start crio.service
```

### Installing crictl

```
VERSION="v1.22.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```
測試指令
```
sudo crictl ps
```

### Installing Kubernetes
#### 設定module

#br_netfilter:bridge可傳送到iptable
#overlay:開啟overlayfs

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```
sudo modprobe overlay
sudo modprobe br_netfilter
```

#### 套用參數

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

`sudo sysctl --system`


#### Firewall
##### kubernetes rule
```
sudo firewall-cmd --zone=public --add-service=kube-apiserver --permanent
sudo firewall-cmd --zone=public --add-service=etcd-client --permanent
sudo firewall-cmd --zone=public --add-service=etcd-server --permanent
#kubelet API
sudo firewall-cmd --zone=public --add-port=10250/tcp --permanent
#kube-scheduler
sudo firewall-cmd --zone=public --add-port=10251/tcp --permanent
#kube-controller-manager
sudo firewall-cmd --zone=public --add-port=10252/tcp --permanent
#NodePort Services
sudo firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
#apply changes
sudo firewall-cmd --reload
```

##### calico rule

```
#BGP
sudo firewall-cmd --zone=public --add-port=179/tcp --permanent
#VXLAN 
sudo firewall-cmd --zone=public --add-port=4789/tcp --permanent
#Typha
sudo firewall-cmd --zone=public --add-port=5473/tcp --permanent
#IPv4 Wireguard enabled
sudo firewall-cmd --zone=public --add-port=51820/tcp --permanent
```

##### flannel rule
##### Master rule
```
firewall-cmd --permanent --add-port=6443/tcp # Kubernetes API server
firewall-cmd --permanent --add-port=2379-2380/tcp # etcd server client API
firewall-cmd --permanent --add-port=10250/tcp # Kubelet API
firewall-cmd --permanent --add-port=10251/tcp # kube-scheduler
firewall-cmd --permanent --add-port=10252/tcp # kube-controller-manager
firewall-cmd --permanent --add-port=8285/udp # Flannel
firewall-cmd --permanent --add-port=8472/udp # Flannel
firewall-cmd --add-masquerade --permanent
#only if you want NodePorts exposed on control plane IP as well
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --reload
systemctl restart firewalld
```


##### Node rule
```
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=8285/udp # Flannel
firewall-cmd --permanent --add-port=8472/udp # Flannel
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
systemctl restart firewalld
```

or

關掉防火牆
```
systemctl disable firewalld
systemctl stop firewalld
```
#### Set SELinux in permissive mode (effectively disabling it)
`setenforce 0`
`sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config`

#### Set swap off

檢查swap:
`free -h`

暫時關閉:
`sudo swapoff /dev/mapper/centos-swap`
or
`sudo swapoff -a `



永久關閉:
註解掉swap 
`vi /etc/fstab`
![](https://i.imgur.com/RCEG9Po.png)

#### Setting kuberentes yum Repo
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

`export KUBE_VERSION="1.22.5"`

`sudo yum install -y kubelet-${KUBE_VERSION} kubeadm-${KUBE_VERSION} kubectl-${KUBE_VERSION} --disableexcludes=kubernetes`


```
sudo systemctl enable --now kubelet
sudo systemctl start kubelet
```


### Init kubernetes
選擇其中一台作為Master

`sudo kubeadm init --pod-network-cidr=192.168.0.0/16`

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

到其他兩台使用init產出的

sudo kubeadm join <IP>:<Port> --token <token> \
    --discovery-token-ca-cert-hash sha256:<num>

如果要讓Master也可以被部屬主要元件之外的pod
`kubectl taint nodes --all node-role.kubernetes.io/master-`
    
    
檢查Node狀態
`kubectl get nodes`
![](https://hackmd.io/_uploads/BkeGsCtw2.png)