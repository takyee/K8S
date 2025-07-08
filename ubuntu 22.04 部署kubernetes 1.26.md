# Ubuntu 22.44部署kubernetes 1.26

## 环境准备

准备工作需要在所有节点上操作，包含的过程如下： 

- 配置主机名
- 添加/etc/hosts
- 清空防火墙
- 设置apt源
- 配置时间同步
- 关闭swap
- 配置内核参数
- 加载ip_vs内核模块
- 安装Containerd
- 安装kubelet、kubectl、kubeadm

#### 修改主机名：

```Bash
# 以一个节点为例
# k8s01
hostnamectl set-hostname k8s01 --static
# k8s02
hostnamectl set-hostname k8s02 --static
# k8s03
hostnamectl set-hostname k8s03 --static
```

#### 添加/etc/hosts:

```Bash
# k8s01
echo "172.26.159.93 k8s01" >> /etc/hosts
# k8s02
echo "172.26.159.94 k8s02" >> /etc/hosts
# k8s03
echo "172.26.159.95 k8s03" >> /etc/hosts

```

#### 清空防火墙规则

```Bash
sudo iptables -F

sudo iptables -t nat -F
```

### 配置时区

```Bash
sudo timedatectl set-timezone Asia/Shanghai
```

#### 配置时间同步： 

```Bash
sudo apt install -y chrony 
sudo systemctl enable --now chronyd 
chronyc sources 
```

#### 关闭swap： 

默认情况下，kubernetes不允许其安装节点开启swap，如果已经开始了swap的节点，建议关闭掉swap

```Bash
# 临时禁用swap
swapoff -a 

# 修改/etc/fstab，将swap挂载注释掉，可确保节点重启后swap仍然禁用

# 可通过如下指令验证swap是否禁用： 
free -m  # 可以看到swap的值为0
              total        used        free      shared  buff/cache   available
Mem:           7822         514         184         431        7123        6461
Swap:             0           0           0

```

#### 关闭防火墙（ufw）
```Bash
sudo systemctl stop ufw.service
sudo systemctl disable ufw.service
```

#### 加载内核模块：

```Bash
cat > /etc/modules-load.d/modules.conf<<EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack;do modprobe $i;done

```

> 这些内核模块主要用于后续将kube-proxy的代理模式从iptables切换至ipvs

#### 修改内核参数：

```Bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf

```

> 如果出现sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No Such file or directory这样的错误，说明没有先加载内核模块br_netfilter。

bridge-nf 使 netfilter 可以对 Linux 网桥上的 IPv4/ARP/IPv6 包过滤。比如，设置`net.bridge.bridge-nf-call-iptables=1`后，二层的网桥在转发包时也会被 iptables的 FORWARD 规则所过滤。常用的选项包括： 

- net.bridge.bridge-nf-call-arptables：是否在 arptables 的 FORWARD 中过滤网桥的 ARP 包
- net.bridge.bridge-nf-call-ip6tables：是否在 ip6tables 链中过滤 IPv6 包
- net.bridge.bridge-nf-call-iptables：是否在 iptables 链中过滤 IPv4 包
- net.bridge.bridge-nf-filter-vlan-tagged：是否在 iptables/arptables 中过滤打了 vlan 标签的包。

#### 安装 containerd：
> 参考 containerd 官方文档 https://github.com/containerd/containerd/blob/main/docs/getting-started.md

##### 配置 apt repository
1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:

```Bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```
2. Add Docker’s official GPG key:

```Bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
3. Use the following command to set up the repository:

```Bash
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Install containerd.

```Bash
sudo apt -y update
sudo apt install containerd.io

# 生成containerd的配置文件

sudo mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# 修改/etc/containerd.config.toml配置文件以下内容： 
......
[plugins]
  ......
  [plugins."io.containerd.grpc.v1.cri"]
    ...
    #sandbox_image = "k8s.gcr.io/pause:3.6"
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
    ...
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
      ...
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true #对于使用 systemd 作为 init system 的 Linux 的发行版，使用 systemd 作为容器的 cgroup driver 可以确保节点在资源紧张的情况更加稳定
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://pqbap4ya.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://registry.aliyuncs.com/google_containers"]  
        ......
        
sudo systemctl enable containerd --now

# 验证
ctr version 
```

5. containerd 配置 Proxy (解决无法下载image的问题)

```Bash
sudo mkdir -p /etc/systemd/system/containerd.service.d
sudo editor /etc/systemd/system/containerd.service.d/proxy-http.conf
# 添加以下内容到配置文件 proxy-http.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80"
Environment="HTTPS_PROXY=http://proxy.example.com:443"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"

sudo systemctl daemon-reload
sudo systemctl restart containerd

# 验证代理是否配置生效
 sudo systemctl show --property=Environment containerd.service
```

#### 安装kubeadm, kubelet, kubectl
> 参考 kubernetes 官方文档 https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

1. 更新 apt 包索引并安装使用 Kubernetes apt 仓库所需要的包：
```Bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
```

2. 下载 Google Cloud 公开签名秘钥：
```Bash
# 国内环境从 google 下载密钥 需要给curl 添加 --proxy 参数
 sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
3. 添加 Kubernetes apt 仓库：
```Bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. 更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：
```Bash
# 因为 kubernetes 配置了google的apt源，使用apt install 导致无法下载安装包，可以为apt配置proxy解决
sudo nano /etc/apt/apt.conf.d/proxy.conf
Acquire::http::Proxy "http://username:password@proxy-server-ip:8181/";
   Acquire::https::Proxy "http://username:password@proxy-server-ip:8182/";
   Acquire::http::Proxy {
       archive.ubuntu.com DIRECT; //请修改为不需要代理的repo源，例如cn.archive.ubuntu.com DIRECT;
       download.docker.com DIRECT; //请修改为不需要代理的repo源，例如download.docker.com DIRECT;
       your2.local.repository DIRECT;//同上
   };

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
>说明：
在低于 Debian 12 和 Ubuntu 22.04 的发行版本中，/etc/apt/keyrings 默认不存在。 如有需要，你可以创建此目录，并将其设置为对所有人可读，但仅对管理员可写。

#### 部署master

部署master，只需要在master节点上配置，包含的过程如下：

- 生成kubeadm-config.yaml文件
- 编辑kubeadm-config.yaml文件
- 根据配置的kubeadm-config.yaml文件部署master

通过如下指令创建默认的kubeadm.yaml文件：
```Bash
kubeadm config print init-defaults   > ~/kubeadm.yaml
```
修改kubeadm.yaml文件如下：
```yaml
apiVersion: kubeadm.k8s.io/v1beta4
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
  advertiseAddress: 10.10.100.5
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  imagePullSerial: true
  name: dgm-svcp01  
  taints: null
timeouts:
  controlPlaneComponentHealthCheck: 4m0s
  discovery: 5m0s
  etcdAPICall: 2m0s
  kubeletHealthCheck: 4m0s
  kubernetesAPICall: 1m0s
  tlsBootstrap: 5m0s
  upgradeManifests: 5m0s
---
apiServer:
  certSANs: # certSANs用于为 Kubernetes API Server 的证书添加额外的主机名或 IP 地址，使得 API Server 的证书在通过这些域名/IP 访问时也能被视为“受信任”。                     
    - dgm-svk8sapi.homag.com.cn
    - dgm-svcp01
    - dgm-svcp02
    - dgm-svcp03
    - 10.10.100.5
    - 10.10.100.6
    - 10.10.13.5
    - 127.0.0.1
apiVersion: kubeadm.k8s.io/v1beta4
controlPlaneEndpoint: "dgm-svk8sapi.homag.com.cn:6443"  ### 高可用控制平面
caCertificateValidityPeriod: 87600h0m0s # kubernetes CA证书过期时间（默认10年）
certificateValidityPeriod: 87600h0m0s # 修改kubernetes服务器和各个组件证书过期时间修改为10年（默认8760h0m0s =1年）
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
encryptionAlgorithm: RSA-2048
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.33.2  # 指定kubernetes版本
networking:
  dnsDomain: cluster.local
  serviceSubnet: 192.168.198.0/23 # service子网
  podSubnet: 192.168.215.0/23  # Pod子网
proxy: {}
scheduler: {}

```
拉取镜像： 

```Bash
sudo kubeadm config images pull --config kubeadm.yaml
```
安装master节点：

```Bash
sudo kubeadm init --config kubeadm.yaml
```
此时如下看到类似如下输出即代表master安装完成： 

```Bash
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.10.100.151:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:320a135a2de65714d760068e2a7c2873dc03b7f089ee32740f8ee3a60c954c1b
```
配置访问集群：

```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
配置kubectl命令自动补全
```Bash
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
source ~/.bashrc # reload bash shell
```

#### 删除kubernetes

```Bash
sudo kubeadm reset -f kubeadm.yaml
# 安装中途出错，可以删除已经安装的组件，进行重新安装
```

#### 配置网络

在master完成部署之后，发现两个问题：

1. master节点一直notready
2. coredns pod一直pending

其实这两个问题都是因为还没有安装网络插件导致的，kubernetes支持众多的网络插件，详情可参考这里： https://kubernetes.io/docs/concepts/cluster-administration/addons/

我们这里使用calico网络插件，安装如下： 
```Bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml -O 
kubectl apply -f calico.yaml
# 无法下载可以 curl --proxy
# 修改calico 分配给pod的地址池大小 **
# 添加以下配置

- name: CACLICO_IPV4POOL_BLOCK_SIZE
  value: "26"
```





>参考：https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises?spm=wolai.workspace.0.0.60726ce4nWuMHU#install-calico-with-kubernetes-api-datastore-50-nodes-or-less

部署完成后，可以通过如下指令验证组件是否正常： 
检查master组件是否正常：

>**注意：切记！切记！切记！**一定要在/etc/systemd/system/containerd.service.d/proxy-http.conf配置中排除pods和service的网段
```Bash
cat /etc/systemd/system/containerd.service.d/proxy-http.conf
[Service]
Environment="HTTP_PROXY=http://10.10.10.8:1083"
Environment="HTTPS_PROXY=http://10.10.10.8:1083"
Environment="NO_PROXY=localhost,127.0.0.1,docker.io,production.cloudflare.docker.com,10.96.0.0/12,10.244.0.0/16"
# 10.96.0.0/12,10.244.0.0/16 是kubeadm.yaml中配置的pod和service的子网
```

```Bash
kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5857bf8d58-8knmc   1/1     Running   0          14m
calico-node-hfhb7                          1/1     Running   0          14m
coredns-787d4945fb-2vd22                   1/1     Running   0          25m
coredns-787d4945fb-d84xq                   1/1     Running   0          25m
etcd-k8s01                                 1/1     Running   0          25m
kube-apiserver-k8s01                       1/1     Running   0          25m
kube-controller-manager-k8s01              1/1     Running   0          25m
kube-proxy-lrt75                           1/1     Running   0          25m
kube-scheduler-k8s01                       1/1     Running   0          25m
```

查看节点状态： 

```Bash
kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
k8s01   Ready    control-plane   26m   v1.26.3
```


### 安装work节点

1. 参照master初始化安装步骤

2. 添加/etc/hosts:
```Bash
sudo nano /etc/hosts
# k8s01
10.10.100.151 k8s01
# k8s02
10.10.100.152 k8s02
# k8s03
10.10.100.153 k8s03
```

#### 清空防火墙规则

```Bash
sudo iptables -F
sudo iptables -t nat -F
```

#### 关闭防火墙（ufw）
```Bash
sudo systemctl stop ufw.service
sudo systemctl disable ufw.service
```

### 配置时区

```Bash
sudo timedatectl set-timezone Asia/Shanghai
```

#### 配置时间同步： 

```Bash
sudo apt install -y chrony 
sudo systemctl enable --now chronyd 
chronyc sources 
```

#### 关闭swap： 
默认情况下，kubernetes不允许其安装节点开启swap，如果已经开始了swap的节点，建议关闭掉swap

```Bash
# 临时禁用swap
swapoff -a 

# 修改/etc/fstab，将swap挂载注释掉，可确保节点重启后swap仍然禁用

# 可通过如下指令验证swap是否禁用： 
free -m  # 可以看到swap的值为0
              total        used        free      shared  buff/cache   available
Mem:           7822         514         184         431        7123        6461
Swap:             0           0           0

```


#### 加载内核模块：

```Bash
cat > /etc/modules-load.d/modules.conf<<EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack;do modprobe $i;done
```

> 这些内核模块主要用于后续将kube-proxy的代理模式从iptables切换至ipvs

#### 修改内核参数：

```Bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf

```

> 如果出现sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No Such file or directory这样的错误，说明没有先加载内核模块br_netfilter。

bridge-nf 使 netfilter 可以对 Linux 网桥上的 IPv4/ARP/IPv6 包过滤。比如，设置`net.bridge.bridge-nf-call-iptables=1`后，二层的网桥在转发包时也会被 iptables的 FORWARD 规则所过滤。常用的选项包括： 

- net.bridge.bridge-nf-call-arptables：是否在 arptables 的 FORWARD 中过滤网桥的 ARP 包
- net.bridge.bridge-nf-call-ip6tables：是否在 ip6tables 链中过滤 IPv6 包
- net.bridge.bridge-nf-call-iptables：是否在 iptables 链中过滤 IPv4 包
- net.bridge.bridge-nf-filter-vlan-tagged：是否在 iptables/arptables 中过滤打了 vlan 标签的包。

#### 安装 containerd：
> 参考 containerd 官方文档 https://github.com/containerd/containerd/blob/main/docs/getting-started.md

##### 配置 apt repository
1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:

```Bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```
2. Add Docker’s official GPG key:

```Bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
3. Use the following command to set up the repository:

```Bash
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Install containerd.

```Bash
sudo apt -y update
sudo apt install containerd.io

# 生成containerd的配置文件

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# 修改/etc/containerd.config.toml配置文件以下内容： 
......
[plugins]
  ......
  [plugins."io.containerd.grpc.v1.cri"]
    ...
    #sandbox_image = "k8s.gcr.io/pause:3.6"
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
    ...
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
      ...
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true #对于使用 systemd 作为 init system 的 Linux 的发行版，使用 systemd 作为容器的 cgroup driver 可以确保节点在资源紧张的情况更加稳定
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://pqbap4ya.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://registry.aliyuncs.com/google_containers"]  
        ......
        
sudo systemctl enable containerd --now

# 验证
ctr version 
```

5. containerd 配置 Proxy (解决)

```Bash
sudo mkdir -p /etc/systemd/system/containerd.service.d
sudo nano /etc/systemd/system/containerd.service.d/proxy-http.conf
# 添加以下内容到配置文件 proxy-http.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80"
Environment="HTTPS_PROXY=http://proxy.example.com:443"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"

sudo systemctl daemon-reload
sudo systemctl restart containerd

# 验证代理是否配置生效
 sudo systemctl show --property=Environment containerd.service
```

#### 安装kubelet, kubeadm, kubectl
> 参考 kubernetes 官方文档 https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

1. 更新 apt 包索引并安装使用 Kubernetes apt 仓库所需要的包：
```Bash
   sudo apt update
   sudo apt install -y apt-transport-https ca-certificates curl
```

2. 下载 Google Cloud 公开签名秘钥：
```Bash
# 国内环境从 google 下载密钥 需要给curl 添加 --proxy 参数
 sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
3. 添加 Kubernetes apt 仓库：
```Bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. 更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：
```Bash
# 因为 kubernetes 配置了google的apt源，使用apt install 导致无法下载安装包，可以为apt配置proxy解决
sudo nano /etc/apt/apt.conf.d/proxy.conf
Acquire::http::Proxy "http://username:password@proxy-server-ip:8181/";
   Acquire::https::Proxy "http://username:password@proxy-server-ip:8182/";
   Acquire::http::Proxy {
       archive.ubuntu.com DIRECT; //请修改为不需要代理的repo源，例如cn.archive.ubuntu.com DIRECT;
       download.docker.com DIRECT; //请修改为不需要代理的repo源，例如download.docker.com DIRECT;
       your2.local.repository DIRECT;//同上
   };

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
>说明：
在低于 Debian 12 和 Ubuntu 22.04 的发行版本中，/etc/apt/keyrings 默认不存在。 如有需要，你可以创建此目录，并将其设置为对所有人可读，但仅对管理员可写。

#### 添加worker节点
```Bash
# node执行以下命令
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>

# 以下命令在控制节点输出 Node加入集群所需要的--token的值
kubeadm token list

#默认Token的值24小时有效，也可用使用下面的命令创建
kubeadm token create

#以下命令在控制节点输出 Node加入集群所需要的-discovery-token-ca-cert-hash 值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null |    openssl dgst -sha256 -hex | sed 's
```
