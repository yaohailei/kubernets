# Kubernetes 1.15部署

[TOC]

## 安装docker

* 安装gcc

```shell
yum -y install gcc
yum -y install gcc-c++
```

* 卸载老版本的docker和其他相关依赖

```shell
yum  remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine
```

* 安装 yum-utils 等组件

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

* 添加docker的yum 源

```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
```

* 查看仓库中docker的版本列表

```shell
yum list docker-ce --showduplicates | sort -r
```

* 选择版本安装，安装最新的稳定版18.09

```shell
yum -y install docker-ce-18.09.8
```

* 设置docker

```shell
#  Create /etc/docker directory.
mkdir /etc/docker
#  Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": [
        "http://hub-mirror.c.163.com"
  ]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
#  Restart Docker
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```

## 设置系统环境

* 关闭swap，编辑/etc/fstab,注释其中swap相关行
* 关闭selinux，编辑/etc/sysconfig/selinux，设置SELINUX=disabled
* 设置各主机hostname，并配置好hosts文件
* 设置好时区并且同步时间
* 关闭系统防火墙

```shell
systemctl stop firewalld
systemctl disable firewalld
```

* 配置各节点系统内核参数使流过网桥的流量也进入iptables/netfilter框架中，在/etc/sysctl.conf中添加以下配置：

```bash
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
#  使配置生效
sysctl -p
```

* 设置iptables转发

```bash
iptables -P FORWARD ACCEPT
```

* 重启系统

## 安装k8s

### 配置yum源

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0

EOF

yum clean all
yum makecache
```

### master节点操作

* master 节点安装,最好指定版本

```bash
yum -y install kubelet-1.15.0 kubeadm-1.15.0 kubectl-1.15.0 --disableexcludes=kubernetes
```

* 启动kubelet并设置开机自启

```bash
systemctl enable kubelet && systemctl start kubelet
```

* 生成初始化安装的配置文件

```bash
kubeadm config print init-defaults > init.default.yaml
```

* 修改配置文件

1.localAPIEndpoint下的advertiseAddress要修改成本机真实ip

2.imageRepository修改成registry.cn-hangzhou.aliyuncs.com/google_containers

3.kubernetesVersion修改成自己之前安装的版本，如之前安装的v1.15.0

4.networking添加podSubnet: 10.224.0.0/16

* 拉取安装需要的镜像

```shell
kubeadm config images pull --config=init.default.yaml
```

* 初始化安装master节点

```bash
kubeadm init --config=init.default.yaml
```

* 安装完成之后按照提示操作,先拷贝配置

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* 安装完成之后会提示node加入集群的方法，如下所示格式

```bash
kubeadm join 10.253.1.115:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f5b5ba24434cbe841566f20a3ed090fa3c955fabfde91847a7164a10eebdec40 
```

### slave节点操作

* 安装工具

```bash
yum -y install kubelet-1.15.0 kubeadm-1.15.0  --disableexcludes=kubernetes
```

* 启动kubelet与设置开机自启

```bash
systemctl enable kubelet && systemctl start kubelet
```

### 安装网络插件

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

