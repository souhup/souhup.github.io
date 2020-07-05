---
title: 搭建utils.pro
date: 2020-07-05 13:48:00
categories:
  - tutorial
tags: 
  - website
  - ECS
  - docker
  - github
  - 镜像仓库
  - 证书
---

由于想要搭建一个可以提供各类基础功能的网站, 所以便立刻行动起来了😆.

### 功能需求

#### 需求1

提供一个查询自己公网IP的功能, 

通过`GET`访问`http(s)://api.utils.pro/v1/myip`, 获取自己的公网IP与其它信息.

获取到的信息将以简单的文本形式进行展示, `IP:xxx.xxx.xxx.xxx, 国家(地区): xxxx, 省份: xxx, 城市: xxx, 运营商: xxx, 时区: xxx `

#### 需求2

提供一个查询任意IP的功能

通过`GET`访问`http(s)://api.utils.pro/v1/query-ip?ip=xxx.xxx.xxx.xxx` 获取自己的公网IP与其它信息.

获取到的信息将以简单的文本形式进行展示, `IP:xxx.xxx.xxx.xxx, 国家(地区): xxxx, 省份: xxx, 城市: xxx, 运营商: xxx, 时区: xxx `

#### 需求3

通过`curl`等HTTP客户端来进行文件互传.

发送方或接收方通过`GET`调用`http(s)://api.utils.pro/v1/data-transfer-code`获取一个随机密码.

通过该随机密码, 发送方通过`PUT`调用`http(s)://api.utils.pro/v1/data-transfer?code=xxx`，

接收方通过`GET`调用`http(s)://api.utils.pro/v1/data-transfer?code=xxx`


### 实现的思路

实现的话, 应该满足以下的条件

* 足够的省钱🤣
* 便于之后的功能扩展
* 程序部署方便
* 考虑系统的安全性

项目开发的选择如下

* 在云厂商购买免费的`单域名`证书. 
* 功能托管将采用的云厂商云服务器.
* 因为要省钱, 所以网关采用服务器内搭建Nginx而非负载均衡器, 等之后再换吧
* 采用云厂商的日志服务
* 代码托管采用`github`的私有仓库
* 软件库采用云厂商私有镜像仓库
* 暂不考虑前端设计
* 服务器安全性

### 购买域名

在某云厂商购买域名`utils.pro`, 并为该域名免费的单域名证书.

之后会考虑let's encrypt提供的免费泛域名证书

### 创建服务器

在某云厂商购买了服务器， 配置公网IP, 并配置域名解析. 

从安全角度: 创建由云厂商托管的服务器登录密钥.

从安全角度: 配置服务器安全组, 外网入网端口仅包含22,80,443

### 配置服务器内容

选择操作系统`centos-8`

安装软件`nginx`, `docker`

### 创建项目

在`github`上创建私有仓库, 用来存放功能设计文档与代码实现.

后台编程语言选择`go`

运营脚本语言选择`python`

因为之前自己买过`intelij `家的全家桶, 所以IDE选择`goland`

业务指标与性能指标采集选择`prometheus`

`prometheus`指标展示采用`grafana`

### 配置镜像仓库

再某云厂商创建私有镜像仓库, 将代码源选择为`github`. 选择代码提交后满足特定TAG即可自动构建.

镜像构建后通过触发器通知云服务器.

编写python脚本用以监听特定端口, 当镜像仓库通知服务器镜像变更后, 服务器即可更新docker容器.