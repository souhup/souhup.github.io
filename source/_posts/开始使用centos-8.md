---
title: 开始使用centos-8
date: 2019-12-08 23:11:24
categories:
  - tutorial
tags: 
  - linux
  - centos-8
  - docker
  - mysql
  - grafana
  - prometheus
---

计划之后将centos-8作为平时的开发环境，记录一下安装过程。

### 关闭防火墙

我发现centos-8自带的防火墙影响了我对本机程序的telnet，所以需要关闭.

```bash
systemctl stop firewalld
systemctl disable firewalld
```

###  更新软件包

```bash
# 更新软件包
dnf upgrade

# 安装基础软件
dnf install -y epel-release
dnf install -y vim telnet tmux
dnf install -y wget iftop htop net-tools bind-utils tcpdump
```

### 安装docker

在安装docker时发现containerd.io版本过低导致报错，所以需要安装新版containerd.io

```bash
# 安装docker repo
dnf config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装docker 
dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
dnf install docker-ce
systemctl enable docker
systemctl start docker

```

### 更换docker仓库地址

```bash
# 更新docker镜像仓库
mkdir -p /etc/docker
cat << EOF > /etc/docker/daemon.json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
EOF
systemctl restart docker
```

### 将普通用户加入docker组

为了避免普通用户运行docker时需要sudo,所以将其加入到docker组中
```bash
# 将当前用户添加到 docker 组
sudo gpasswd -a ${USER} docker
# 随后用户需要注销并重新登录
```

### 安装docker版mysql

```bash
# 通过docker安装mysql-8
docker run \
    --name mysql \
    -d \
    -e MYSQL_ROOT_PASSWORD=123456 \
    mysql
docker cp mysql:/var/lib/mysql /var/lib/mysql
docker cp mysql:/etc/mysql /etc/mysql
docker stop mysql
docker rm mysql
docker run \
    --name mysql \
    --restart=always \
    -d \
    -p 3306:3306 \
    --user root \
    -v /etc/mysql:/etc/mysql \
    -v /var/lib/mysql:/var/lib/mysql \
    mysql
```

### 安装docker版grafana

```bash
# 通过docker安装grafana
docker run \
    --name=grafana \
    -d \
    --user root \
    grafana/grafana
docker cp grafana:/etc/grafana /etc/grafana
docker stop grafana
docker rm grafana
docker run \
    --name=grafana \
    --restart=always \
    -d -p 3000:3000 \
    --volume /var/lib/grafana:/var/lib/grafana \
    --volume /etc/grafana:/etc/grafana \
    --user root \
    grafana/grafana
```

### 安装docker版prometheus

```bash
# 通过docker安装prometheus
docker run \
    --name=prometheus \
    -d \
    --user root \
    prom/prometheus
docker cp prometheus:/etc/prometheus/prometheus.yml /app/prometheus/config
docker stop prometheus
docker rm prometheus
docker run \
    --name=prometheus \
    --restart=always \
    -d -p 9090:9090 \
    --volume /app/prometheus/config:/etc/prometheus \
    --volume /app/prometheus/data:/prometheus \
    --user root \
    prom/prometheus
```
