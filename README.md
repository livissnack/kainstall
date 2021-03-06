# kainstall  =  kubeadm install kubernetes

使用 shell 脚本, 基于 kubeadm 一键部署 kubernetes 集群



## 为什么？

**为什么要搞这个？Ansible PlayBook 不好么？**

**因为懒**，Ansible PlayBook 编排是非常给力的，不过需要安装 Python 和 Ansible, 且需要下载多个 yaml 文件 。**因为懒**，我想要个更简单的方式来**快速部署**一个分布式的 **Kubernetes HA** 集群， 使用 **shell** 脚本可以不借助外力直接在服务器上运行，省时省力。 并且 shell 脚本只有一个文件，文件大小**不到 100 KB**，非常小巧，可以实现一条命令安装集群的超快体验，而且配合**离线安装包**，可以在不联网的环境下安装集群，这体验真的**非常爽**啊。



## 要求

OS: `centos 7.x x64` , `centos 8.x x64`

CPU: `2C`

MEM: `4G`

认证: 集群节点需**统一认证**; 使用密码认证时，集群节点需使用同一用户名和密码，使用密钥认证时，集群节点需使用同一个密钥文件登陆。

> 未指定离线包时，需要连通外网，用于下载 kube 组件和 docker 镜像。



## 架构



![](./images/k8s-node-ha.png)

> 如需按照步骤安装集群，可参考 https://lework.github.io/2019/10/01/kubeadm-install/



## 功能

- 服务器初始化。
  - 关闭 `selinux`
  - 关闭 `swap`
  - 关闭 `firewalld`
  - 关闭大内存页
  - 配置 `epel` 源
  - 修改 `limits`
  - 配置内核参数
  - 配置 `history` 记录
  - 配置 `journal` 日志
  - 配置 `chrony`时间同步
  - 安装 `ipvs` 模块
  - 更新内核
- 安装`docker`, `kube`组件。
- 初始化`kubernetes`集群,以及增加或删除节点。
- 安装`ingress`组件，可选`nginx`，`traefik`。
- 安装`network`组件，可选`flannel`，`calico`， 需在初始化时指定。
- 安装`monitor`组件，可选`prometheus`。
- 安装`log`组件，可选`elasticsearch`。
- 安装`storage`组件，可选`rook`，`longhorn`。
- 安装`web ui`组件，可选`dashboard`, `kubesphere`。
- 升级到`kubernetes`指定版本。
- 更新集群证书。
- 添加运维操作，如备份etcd快照。
- 支持**离线部署**。
- 支持**sudo特权**。
- 支持**10年证书期限**。

## 默认版本


| 分类                                           | 软件                                             | kainstall 默认版本 | 软件最新版本                                                 |
| ------------------------------------------------ | ------------------ | ------------------------------------------------------------ | ------------------------------------------------ |
| common | [docker-ce](https://github.com/docker/docker-ce) | latest             | ![docker-ce release](https://img.shields.io/github/v/release/docker/docker-ce?sort=semver) |
| common | [kubernetes](https://github.com/kubernetes/kubernetes) | latest             | ![kubernetes release](https://img.shields.io/github/v/release/kubernetes/kubernetes?sort=semver) |
| network | [flannel](https://github.com/coreos/flannel) | 0.13.0            | ![flannel release](https://img.shields.io/github/v/release/coreos/flannel) |
| network | [calico](https://github.com/projectcalico/calico) | 3.16.3 | ![calico release ](https://img.shields.io/github/v/release/projectcalico/calico?sort=semver) |
| addons | [metrics server](https://github.com/kubernetes-sigs/metrics-server) | 0.3.7             | ![metrics-server release](https://img.shields.io/github/v/release/kubernetes-sigs/metrics-server) |
| ingress | [ingress nginx controller](https://github.com/kubernetes/ingress-nginx) | 0.40.2            | ![ingress-nginx release](https://img.shields.io/github/v/release/kubernetes/ingress-nginx?sort=semver) |
| ingress | [traefik](https://github.com/traefik/traefik) | 2.3.2            | ![traefik release ](https://img.shields.io/github/v/release/traefik/traefik?sort=semver) |
| monitor | [kube_prometheus](https://github.com/prometheus-operator/kube-prometheus) | 0.6.0             | ![kube-prometheus release](https://img.shields.io/github/v/release/prometheus-operator/kube-prometheus) |
| log | [elasticsearch](https://github.com/elastic/elasticsearch) | 7.9.2             | ![elasticsearch release](https://img.shields.io/github/v/release/elastic/elasticsearch?sort=semver) |
| storage | [rook](https://github.com/rook/rook) | 1.4.6 | ![rook release](https://img.shields.io/github/v/release/rook/rook?sort=semver) |
| storage | [longhorn](https://github.com/longhorn/longhorn) | 1.0.2 | ![longhorn release](https://img.shields.io/github/v/release/longhorn/longhorn?sort=semver) |
| ui | [kubernetes_dashboard](https://github.com/kubernetes/dashboard) | 2.0.4             | ![kubernetes dashboard release](https://img.shields.io/github/v/release/kubernetes/dashboard?sort=semver) |
| ui | [kubesphere](https://github.com/kubesphere/kubesphere) | 3.0.0            | ![kubesphere release](https://img.shields.io/github/v/release/kubesphere/kubesphere?sort=semver) |


除 **kube组件** 版本可以通过参数(`--version`) 指定外，其他的软件版本需在脚本中指定。



## 使用

> 案例使用请见：[https://lework.github.io/2020/09/26/kainstall](https://lework.github.io/2020/09/26/kainstall)

### 下载脚本

```bash
wget https://cdn.jsdelivr.net/gh/lework/kainstall/kainstall.sh
```

### 帮助信息

```bash
# bash kainstall.sh 

Install kubernetes cluster using kubeadm.

Usage:
  kainstall.sh [command]

Available Commands:
  init            Init Kubernetes cluster.
  reset           Reset Kubernetes cluster.
  add             Add nodes to the cluster.
  del             Remove node from the cluster.
  upgrade         Upgrading kubeadm clusters.
  renew-cert      Renew all available certificates.

Flag:
  -m,--master          master node, default: ''
  -w,--worker          work node, default: ''
  -u,--user            ssh user, default: root
  -p,--password        ssh password
     --private-key     ssh private key
  -P,--port            ssh port, default: 22
  -v,--version         kube version, default: latest
  -n,--network         cluster network, choose: [flannel,calico], default: flannel
  -i,--ingress         ingress controller, choose: [nginx,traefik], default: nginx
  -ui,--ui             cluster web ui, choose: [dashboard,kubesphere], default: dashboard
  -M,--monitor         cluster monitor, choose: [prometheus]
  -l,--log             cluster log, choose: [elasticsearch]
  -s,--storage         cluster storage, choose: [rook,longhorn]
  -U,--upgrade-kernel  upgrade kernel
  -of,--offline-file   specify the offline package file to load
      --10years        the certificate period is 10 years.
      --sudo           sudo mode
      --sudo-user      sudo user
      --sudo-password  sudo user password

Example:
  [init cluster]
  kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134,192.168.77.135 \
  --user root \
  --password 123456 \
  --version 1.19.3

  [reset cluster]
  kainstall.sh reset \
  --user root \
  --password 123456

  [add node]
  kainstall.sh add \
  --master 192.168.77.140,192.168.77.141 \
  --worker 192.168.77.143,192.168.77.144 \
  --user root \
  --password 123456 \
  --version 1.19.3

  [del node]
  kainstall.sh del \
  --master 192.168.77.140,192.168.77.141 \
  --worker 192.168.77.143,192.168.77.144 \
  --user root \
  --password 123456
 
  [other]
  kainstall.sh renew-cert --user root --password 123456
  kainstall.sh upgrade --version 1.19.3 --user root --password 123456
  kainstall.sh add --ingress traefik
  kainstall.sh add --monitor prometheus
  kainstall.sh add --log elasticsearch
  kainstall.sh add --storage rook
  kainstall.sh add --ui dashboard
```

### 初始化集群

```bash
# 使用脚本参数
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3

# 使用环境变量
export MASTER_NODES="192.168.77.130,192.168.77.131,192.168.77.132"
export WORKER_NODES="192.168.77.133,192.168.77.134"
export SSH_USER="root"
export SSH_PASSWORD="123456"
export SSH_PORT="22"
export KUBE_VERSION="1.19.3"
bash kainstall.sh init
```
> 默认情况下，除了初始化集群外，还会安装 `ingress: nginx` , `ui: dashboard` 两个组件。

还可以使用一键安装方式, 连下载都省略了。

```bash
bash -c "$(curl -sSL https://cdn.jsdelivr.net/gh/lework/kainstall/kainstall.sh)"  \
  - init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3
```

### 增加节点

> 操作需在 k8s master 节点上操作，ssh连接信息非默认时请指定

```bash
# 增加单个master节点
bash kainstall.sh add --master 192.168.77.135

# 增加单个worker节点
bash kainstall.sh add --worker 192.168.77.134

# 同时增加
bash kainstall.sh add --master 192.168.77.135,192.168.77.136 --worker 192.168.77.137,192.168.77.138
```

### 删除节点

> 操作需在 k8s master 节点上操作，ssh连接信息非默认时请指定
```bash
# 删除单个master节点
bash kainstall.sh del --master 192.168.77.135

# 删除单个worker节点
bash kainstall.sh del --worker 192.168.77.134

# 同时删除
bash kainstall.sh del --master 192.168.77.135,192.168.77.136 --worker 192.168.77.137,192.168.77.138
```

### 重置集群

```bash
bash kainstall.sh reset \
  --user root \
  --password 123456 \
  --port 22 \
```

### 其他操作

> 操作需在 k8s master 节点上操作，ssh连接信息非默认时请指定

**注意：** 添加组件时请保持节点的内存和cpu至少为`2C4G`的空闲。否则会导致节点下线且服务器卡死。

```bash
# 添加 nginx ingress
bash kainstall.sh add --ingress nginx

# 添加 prometheus
bash kainstall.sh add --monitor prometheus

# 添加 elasticsearch
bash kainstall.sh add --log elasticsearch

# 添加 rook
bash kainstall.sh add --storage rook

# 升级版本
bash kainstall.sh upgrade --version 1.19.3

# 重新颁发证书
bash kainstall.sh renew-cert
```

### 默认设置

**注意:** 以下变量都在脚本文件的`environment configuration`部分。可根据需要自行修改，或者为变量设置同名的**环境变量**修改其默认内容。

```bash
# 版本
DOCKER_VERSION="${DOCKER_VERSION:-latest}"
KUBE_VERSION="${KUBE_VERSION:-latest}"
FLANNEL_VERSION="${FLANNEL_VERSION:-0.13.0}"
METRICS_SERVER_VERSION="${METRICS_SERVER_VERSION:-0.3.7}"
INGRESS_NGINX="${INGRESS_NGINX:-0.40.2}"
TRAEFIK_VERSION="${TRAEFIK_VERSION:-2.3.2}"
CALICO_VERSION="${CALICO_VERSION:-3.16.3}"
KUBE_PROMETHEUS_VERSION="${KUBE_PROMETHEUS_VERSION:-0.6.0}"
ELASTICSEARCH_VERSION="${ELASTICSEARCH_VERSION:-7.9.2}"
ROOK_VERSION="${ROOK_VERSION:-1.4.6}"
LONGHORN_VERSION="${LONGHORN_VERSION:-1.0.2}"
KUBERNETES_DASHBOARD_VERSION="${KUBERNETES_DASHBOARD_VERSION:-2.0.4}"
KUBESPHERE_VERSION="${KUBESPHERE_VERSION:-3.0.0}"

# 集群配置
KUBE_APISERVER="${KUBE_APISERVER:-apiserver.cluster.local}"
KUBE_POD_SUBNET="${KUBE_POD_SUBNET:-10.244.0.0/16}"
KUBE_SERVICE_SUBNET="${KUBE_SERVICE_SUBNET:-10.96.0.0/16}"
KUBE_IMAGE_REPO="${KUBE_IMAGE_REPO:-registry.aliyuncs.com/k8sxio}"
KUBE_NETWORK="${KUBE_NETWORK:-flannel}"
KUBE_INGRESS="${KUBE_INGRESS:-nginx}"
KUBE_MONITOR="${KUBE_MONITOR:-prometheus}"
KUBE_STORAGE="${KUBE_STORAGE:-rook}"
KUBE_LOG="${KUBE_LOG:-elasticsearch}"
KUBE_UI="${KUBE_UI:-dashboard}"

# 定义的master和worker节点地址，以逗号分隔
MASTER_NODES="${MASTER_NODES:-}"
WORKER_NODES="${WORKER_NODES:-}"

# 定义在哪个节点上进行设置
MGMT_NODE="${MGMT_NODE:-127.0.0.1}"

# 节点的连接信息
SSH_USER="${SSH_USER:-root}"
SSH_PASSWORD="${SSH_PASSWORD:-}"
SSH_PRIVATE_KEY="${SSH_PRIVATE_KEY:-}"
SSH_PORT="${SSH_PORT:-22}"
SUDO_USER="${SUDO_USER:-root}"

# 节点设置
HOSTNAME_PREFIX="${HOSTNAME_PREFIX:-k8s}"

# 脚本设置
GITHUB_PROXY="${GITHUB_PROXY:-https://gh.lework.workers.dev/}"
```

### 离线部署

**注意**

脚本执行的宿主机上，需要安装 `tar` 命令，用于解压离线包。

> 详细部署请见: [https://lework.github.io/2020/10/18/kainstall-offline/](https://lework.github.io/2020/10/18/kainstall-offline/)


**下载指定版本的离线包**

```bash
wget http://kainstall.oss-cn-shanghai.aliyuncs.com/1.19.3/centos7.tgz
```
> 更多离线包信息，见 [kainstall-offline](https://github.com/lework/kainstall-offline) 仓库


**初始化集群**

> 指定 `--offline-file` 参数。

```bash
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --offline-file centos7.tgz 
```

**添加节点**

> 指定 --offline-file 参数。

```bash
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --offline-file centos7.tgz
```

### sudo 特权

创建 sudo 用户
```bash
useradd test
passwd test --stdin <<< "12345678"
echo 'test    ALL=(ALL)   ALL' >> /etc/sudoers
```

sudo 参数
- `--sudo` 开启 sudo 特权
- `--sudo-user` 指定 sudo 用户, 默认是 `root`
- `--sudo-password` 指定 sudo 密码

示例
```bash
# 初始化
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user test \
  --password 12345678 \
  --port 22 \
  --version 1.19.3 \
  --sudo \
  --sudo-user root \
  --sudo-password 12345678

# 添加
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user test \
  --password 12345678 \
  --port 22 \
  --version 1.19.3 \
  --sudo \
  --sudo-user root \
  --sudo-password 12345678
```

### 10年证书期限

**注意:** 此操作需要联网下载。

使用 [kubeadm-certs](https://github.com/lework/kubeadm-certs) 项目编译的 `kubeadm` 客户端， 其修改了 `kubeadm` 源码，将 1 年期限修改成 10 年期限，具体信息见仓库介绍。

在初始化或添加时，加上 `--10years` 参数，就可以使用`kubeadm` 10 years 的客户端

示例
```bash
# 初始化
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3 \
  --10years
  
# 添加
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3 \
  --10years
```

## 联系方式

- [QQ群](https://qm.qq.com/cgi-bin/qm/qr?k=HwpkLUcmroLKNv37TlrHY-D3SXuLKMOd&jump_from=webapi)

## License

MIT
