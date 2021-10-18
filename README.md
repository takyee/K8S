# CentOS 8 K8S 安装步骤

## 一、CentOS

1.选择最小化安装

2.启用EPEL源

```
yum install epel-release
```

3.启用ELRepo源

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm -y
```

4.安装长期支持版本Kernel

```
dnf --enablerepo=elrepo-kernel install kernel-lt
```

4.更新CentOS

```
yum update -y
```

## 二、Docker

1.Set up the repository

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

2.Install Docker Engine

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

3.Start Docker

```
sudo systemctl start docker
```

## 三、Kubernetes
