# CentOS8 &Ubuntu  K8s 安装步骤

## 一、CentOS

#### 1.选择 最小化安装

#### 2.启用EPEL源

```
yum install epel-release
```

#### 3.启用ELRepo源

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm -y
```

#### 4.安装长期支持版本Kernel

```
dnf --enablerepo=elrepo-kernel install kernel-lt
```

#### 5.Enabling TCP BBR in CentOS

Open the following configuration file `vi /etc/sysctl.conf` to enable enable TCP BBR.

```
vi /etc/sysctl.conf
```

At the end of the config file, add the following lines.

```
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
```

Save the file, and refresh your configuration by using this command,

```
sysctl -p
```

#### 6.更新CentOS

```
yum update -y
```



## 二、Ubuntu 20.04

1.选择 最小化安装

2.升级系统

```
sudo apt update
sudo apt upgrade
```

3.Enabling TCP BBR in Ubuntu

Open the following configuration file `vi /etc/sysctl.conf` to enable enable TCP BBR.

```
vi /etc/sysctl.conf
```

At the end of the config file, add the following lines.

```
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
```

Save the file, and refresh your configuration by using this command,

```
sysctl -p

```

## 三、Docker

#### 	1.Set up the repository (for CentOS)

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 	1.1.Set up the repository (for Ubuntu)

##### 		Uninstall old versions

​		Older versions of Docker were called `docker`, `docker.io`, or `docker-engine`. If these are installed, uninstall them:

```
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

##### 		Set up the repository

1. ###### Update the `apt` package index and install packages to allow `apt` to use a repository over HTTPS:

   ```
   $ sudo apt-get update
   
   $ sudo apt-get install \
       ca-certificates \
       curl \
       gnupg \
       lsb-release
   ```

2. ###### Add Docker’s official GPG key:

   ```
   $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg	
   ```

​	Use the following command to set up the **stable** repository. To add the **nightly** or **test** repository, add the word `nightly` or `test` (or both) after the word `stable` in the commands below. [Learn about **nightly** and **test** channels](https://docs.docker.com/engine/install/).

```
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update the `apt` package index, and install the *latest version* of Docker Engine and containerd, or go to the next step to install a specific version:

Install Docker Engine (for CentOS)

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

#### 	3.Start Docker (for CentOS)

```
sudo systemctl start docker
```

#### 	4.HTTP/HTTPS proxy (可选项)

​	1).Create a systemd drop-in directory for the docker service:

```
sudo mkdir -p /etc/systemd/system/docker.service.d
```

​	2).Create a file named `/etc/systemd/system/docker.service.d/http-proxy.conf` that adds the `HTTP_PROXY` environment variable:

```
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80"
Environment="HTTPS_PROXY=http://proxy.example.com:443"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"
```

​	3).Flush changes and restart Docker

```
 sudo systemctl daemon-reload
 sudo systemctl restart docker
```

​	4).Verify that the configuration has been loaded and matches the changes you made, for example:

```
sudo systemctl show --property=Environment docker
```

#### 	5.Kubernetes 国内镜像

```
registry.aliyuncs.com/google_containers
```

#### 6.Configuring a cgroup driver

```
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

#### 7.Restart Docker and enable on boot:

```shell
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 四、Kubernetes

#### 1.Letting iptables see bridged traffic

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### 2.Installing kubeadm, kubelet and kubectl (For CentOS )

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

**注意：因为安装Kubernetes需要从Google获取安装源，可以修改 /etc/yum.repos.d/kubernetes.repo，在文件添加 proxy=http://user:password@proxy.example.com:3128，即可单独为这个yum源添加代理。**

##### 	For Ubuntu:

1. ##### Update the `apt` package index and install packages needed to use the Kubernetes `apt` repository:

   ```shell
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   ```

2. ##### Download the Google Cloud public signing key:

   ```shell
   sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
   ```

   注意：curl 可以添加 --socks5 或者 proxy 参数，指定代理服务器下载

3. ##### Add the Kubernetes `apt` repository:

   ```shell
   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

4. ##### Setup Proxy for APT

   You will need to set up a proxy for APT if you want to install the package from the Ubuntu repository. You can do it by creating a new configuration file at /etc/apt/apt.conf.d/:

   ```
   nano /etc/apt/apt.conf.d/proxy.conf
   ```

   Add the following lines:

   ```
   Acquire::http::Proxy "http://username:password@proxy-server-ip:8181/";
   Acquire::https::Proxy "http://username:password@proxy-server-ip:8182/";
   Acquire::http::Proxy {
       your.local.repository DIRECT; //请修改为不需要代理的repo源，例如cn.archive.ubuntu.com DIRECT;
       your1.local.repository DIRECT;//请修改为不需要代理的repo源，例如download.docker.com DIRECT;
       your2.local.repository DIRECT;//同上
   };
   ```

   

   Save and close the file when you are finished. You can now install any packages from the Ubuntu repository in your system

5. ##### Update `apt` package index, install kubelet, kubeadm and kubectl, and pin their version:

   ```shell
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```



#### 3.Linux 系统中的 bash 自动补全功能

```
yum install bash-completion
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

#### 4.获取kubeadm 默认配置文件

```
kubeadm config print init-defaults > init.yaml

```

修改init.yaml 中的 advertiseAddress 字段为 master 服务器的IP 地址

#### 5.准备kubeadm config image镜像

​	由于k8s.gcr.io 在大陆无法使用，registry.aliyuncs.com/google_containers 替代

​	a. 可以先使用 kubeadm config images list 查看 kubeadm 需要的镜像

​	b. 使用 docker pull registry.aliyuncs.com/google_containers/ xx_a.b.c.d 拉取所有的镜像

​	c. 使用ducker tag  修改 拉取回来的镜像的TAG为 k8s.gcr.io/xxx.x.x.xabc

#### 6.Creating a cluster with kubeadm

```
kubeadm init --config init.yaml
```

## 五、Calico

#### 1.Install Calico networking and network policy for on-premises deployments

```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```



### 六、Node加入集群

##### Node加入集群的语法：

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

##### 以下命令在控制节点输出 Node加入集群所需要的--token的值

```bash
kubeadm token list
```

##### 默认Token的值24小时有效，也可用使用下面的命令创建

```
kubeadm token create
```

##### 以下命令在控制节点输出 Node加入集群所需要的-discovery-token-ca-cert-hash 值

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null |    openssl dgst -sha256 -hex | sed 's/^.* //'
```



