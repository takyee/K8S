# CentOS 8 K8S 安装步骤

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

#### 4.更新CentOS

```
yum update -y
```

## 二、Docker

#### 1.Set up the repository

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 2.Install Docker Engine

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

#### 3.Start Docker

```
sudo systemctl start docker
```

#### 4.HTTP/HTTPS proxy (可选项)

1).Create a systemd drop-in directory for the docker service:

```
sudo mkdir -p /etc/systemd/system/docker.service.d
```

2).Create a file named `/etc/systemd/system/docker.service.d/http-proxy.conf` that adds the `HTTP_PROXY` environment variable:

```
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80"
Environment="HTTPS_PROXY=http://proxy.example.com:443"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"
```

3).Flush changes and restart Docker

```
 sudo systemctl daemon-reload
 sudo systemctl restart docker
```

4).Verify that the configuration has been loaded and matches the changes you made, for example:

```
sudo systemctl show --property=Environment docker
```

#### 5.Kubernetes 国内镜像

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

## 三、Kubernetes

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

#### 2.Installing kubeadm, kubelet and kubectl

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

#### 3.获取kubeadm 默认配置文件

```
kubeadm config print init-defaults > init.yaml

```

修改init.yaml 中的 advertiseAddress 字段为 master 服务器的IP 地址

#### 4.准备kubeadm config image镜像

​	由于k8s.gcr.io 在大陆无法使用，registry.aliyuncs.com/google_containers 替代

​	a. 可以先使用 kubeadm config images list 查看 kubeadm 需要的镜像

​	b. 使用 docker pull registry.aliyuncs.com/google_containers/ xx_a.b.c.d 拉取所有的镜像

​	c. 使用ducker tag  修改 拉取回来的镜像的TAG为 k8s.gcr.io/xxx.x.x.xabc

#### 5.Creating a cluster with kubeadm

```
kubeadm init --config init.yaml
```

## 四、Calico

#### 1.Install Calico networking and network policy for on-premises deployments

```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

