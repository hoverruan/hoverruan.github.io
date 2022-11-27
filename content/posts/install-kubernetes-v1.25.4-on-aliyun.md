---
title: "在阿里云上安装 Kubernetes v1.25.4"
date: 2022-11-26T22:30:37+08:00
slug: 2022/11/26/install-kubernetes-v1.25.4-on-aliyun
tags:
- kubernetes
- 阿里云
---

本次测试安装最新的发布版本 v1.25.4。比较大的改动是从 v1.24 开始正式 [移除了对dockershim的支持](https://kubernetes.io/blog/2022/01/07/kubernetes-is-moving-on-from-dockershim/)，本文使用 [containerd](https://github.com/containerd/containerd) + [nerdctl](https://github.com/containerd/nerdctl) 来替代原来的 docker。

之前操作系统都选择 CentOS 7.x，但是 7.x 的维护期也只是到2024年6月30日就结束了。后续的服务器操作系统打算切换到阿里云自己维护的 `Alibaba Cloud Linux` 上面，本次测试采用的服务器版本是 `Alibaba Cloud Linux 3.2104`。

本次测试采取在墙外制作系统镜像、在墙内使用的策略，避免配置其他第三方的安装源：

- 创建一台香港的ECS服务器
  - 安装必要的系统软件
  - 拉取 Kubernetes 集群所需的镜像
  - 制作成阿里云的系统自定义镜像
- 将镜像复制到国内的区域
- 在国内区域基于上述镜像，配置 Kubernetes 集群

## 制作自定义镜像

基本思路：

- 通过 nerdctl 的 [full distribution 发行版本](https://github.com/containerd/nerdctl/releases)，完成 `runc`、`containerd`、`CNI plugins` 等重要组件的安装，大大简化了安装过程
- 对 `containerd` 的配置改动有：
  - 默认的 `sandbox_image` 是 `k8s.gcr.io/pause:3.6`，换成了 `registry.k8s.io/pause:3.8`，跟 `kubeadm config images pull` 获取的版本保持一致
  - 通过 `systemd` 来启动
- 执行 `yum install` 时指定具体的版本号，减少不确定性
- 采用 `calico` 作为网络插件

通过下面的脚本，可以一键安装好所需的全部工具并拉取所有的镜像：

```shell
#!/usr/bin/env bash

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

swapoff -a

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

wget https://github.com/containerd/nerdctl/releases/download/v1.0.0/nerdctl-full-1.0.0-linux-amd64.tar.gz
tar Cxzvvf /usr/local nerdctl-full-1.0.0-linux-amd64.tar.gz

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's#k8s.gcr.io/pause:3.6#registry.k8s.io/pause:3.8#g' /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

systemctl daemon-reload
systemctl enable --now containerd

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

yum install -y kubelet-1.25.4-0 kubeadm-1.25.4-0 kubectl-1.25.4-0 --disableexcludes=kubernetes
systemctl enable --now kubelet

kubeadm config images pull

nerdctl -n k8s.io pull docker.io/calico/cni:v3.24.5
nerdctl -n k8s.io pull docker.io/calico/node:v3.24.5
nerdctl -n k8s.io pull docker.io/calico/kube-controllers:v3.24.5

curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/calico.yaml -O
```

安装好之后，基于这台ECS创建一个自定义镜像，然后将此镜像复制到国内区域。

## 使用镜像搭建 Kubernetes 单节点集群

基于上面创建的镜像，在国内的区域也可以很方便地使用 `kubeadm` 创建新集群，因为所需的工具和镜像都已经准备好了。

下面是创建单节点集群的命令：

```shell
kubeadm init --pod-network-cidr=192.168.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.25.4

export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f calico.yaml

# 允许在 master 节点运行普通的业务
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 使用镜像搭建 Kubernetes 多节点集群

Kubernetes 多节点集群通常需要部署至少 3 个 `master` 节点（也叫`control plane nodes`），需要一个负载均衡的地址来将请求分发到各个 `apiserver`：

![Stacked etcd topology](https://d33wubrfki0l68.cloudfront.net/d1411cded83856552f37911eb4522d9887ca4e83/b94b2/images/kubeadm/kubeadm-ha-topology-stacked-etcd.svg)

使用阿里云的 `PrivateZone` 服务，可以比较简单地配置内网的域名。在本次测试中，先配置内网域名 `k8smaster.k8s-in` 作为控制平面的地址。

- 基于镜像创建第一台 master 节点的服务器，命名为 `k8s-master1`
- 在 `PrivateZone` 中添加一条 A 记录，指向 `k8s-master1` 的内网IP地址
- 初始化时通过 `--upload-certs` 参数，将证书存到集群中，后面在添加其他节点时，需要用到证书

初始化命令：

```shell
kubeadm init --control-plane-endpoint=k8smaster.k8s-in \
  --pod-network-cidr=192.168.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.25.4 \
  --upload-certs

export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f calico.yaml
```

初始化的输出内容类似于这样：

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8smaster.k8s-in:6443 --token xnbtmw.bkb0epe6uiof6g3b \
  --discovery-token-ca-cert-hash sha256:82a830b608bf05f71867a182f5f305bda0c6ee447a1aa32ab9c1cd43af217851 \
  --control-plane --certificate-key c2f2d167ea71319c50a1818750f3b9818ae972a572ca7be48ab370c37b9774c7

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8smaster.k8s-in:6443 --token xnbtmw.bkb0epe6uiof6g3b \
  --discovery-token-ca-cert-hash sha256:82a830b608bf05f71867a182f5f305bda0c6ee447a1aa32ab9c1cd43af217851
```

通过 `kubectl get nodes` 确认第一个节点的状态是否 Ready：

```
NAME          STATUS   ROLES           AGE     VERSION
k8s-master1   Ready    control-plane   21s     v1.25.4
```

基于镜像创建第二台 master 节点的服务器，命名为 `k8s-master2`，执行加入集群操作。其中 `--control-plane` 参数表明要以 master 的身份加入集群：

```shell
kubeadm join k8smaster.k8s-in:6443 \
  --token xnbtmw.bkb0epe6uiof6g3b \
  --discovery-token-ca-cert-hash sha256:82a830b608bf05f71867a182f5f305bda0c6ee447a1aa32ab9c1cd43af217851 \
  --control-plane \
  --certificate-key c2f2d167ea71319c50a1818750f3b9818ae972a572ca7be48ab370c37b9774c7
```

通过 `kubectl get nodes` 确认第二个节点的状态是否 Ready：

```
NAME          STATUS   ROLES           AGE     VERSION
k8s-master1   Ready    control-plane   2m57s   v1.25.4
k8s-master2   Ready    control-plane   33s     v1.25.4
```

在 `PrivateZone` 中添加一条 A 记录，指向 `k8s-master2` 的内网IP地址。

最后是添加 worker 节点的命令：

```shell
kubeadm join k8smaster.k8s-in:6443 \
  --token xnbtmw.bkb0epe6uiof6g3b \
  --discovery-token-ca-cert-hash sha256:82a830b608bf05f71867a182f5f305bda0c6ee447a1aa32ab9c1cd43af217851
```

