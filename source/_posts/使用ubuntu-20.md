---
title: 使用ubuntu-20
date: 2020-04-25 20:16:24
categories:
  - tutorial
tags:
  - linux
  - ubuntu-20
  - zsh
  - git
---

# 使用ubuntu-20

## 安装基础工具

```bash
sudo apt update
sudo apt install -y vim git telnet curl wget tmux gitk
sudo apt install -y iftop htop net-tools tcpdump
sudo apt install -y gcc g++ make
```

## 配置本地代理

### 在host中配置本地代理地址

```bash
sudo sed -i '$a192.168.152.1\tmyproxy' /etc/hosts
```

### terminal代理

```bash
# 会话级别代理
export ALL_PROXY="socks5://myproxy:1080"
# 命令级别代理
ALL_PROXY="socks5://myproxy:1080" {其他命令}
```

## 配置git

### 配置基本信息

```bash
git config --global user.name "souhup"
git config --global user.email "souhup@gmail.com"
git config --global core.editor "vim"
git config --global core.quotepath false # git status显示中文
```

### 生成公私钥

```bash
ssh-keygen -t rsa -C ”souhup@gmail.com”
```

### 为github配置代理

```bash
cat >> ~/.ssh/config << EOF
Host github.com
   HostName github.com
   User git
   ProxyCommand nc -v -x myproxy:1080 %h %p
EOF
```

## 安装zsh

### 安装zsh

```bash
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### 安装插件 zsh-autosuggestions

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

### 安装插件 zsh-syntax-highlighting

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### 安装插件autojump

```bash
sudo apt install autojump
```

### 修改.zshrc

```bash
sed -i '/^plugins=/ c plugins=(git zsh-syntax-highlighting autojump zsh-autosuggestions)' ~/.zshrc
```

### 从github上下载自定义的zsh定义

```bash
ALL_PROXY="socks5://myproxy:1080" curl https://raw.githubusercontent.com/souhup/configuration/master/.zshrc >> .zshrc
source ~/.zshrc
```

## 其他

### 后台执行特定软件

```bash
nohup XXX >> /dev/null 2>&1 &
```

