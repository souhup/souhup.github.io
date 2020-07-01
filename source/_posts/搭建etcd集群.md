---
title: 单机搭建etcd集群
date: 2020-07-01 23:56:00
categories:
  - tutorial
tags:
  - etcd
---

在测试时, 有时需在单机搭建etcd集群, 现将步骤记录如下

```bash
#!/bin/bash
docker stop etcd-1 etcd-2 etcd-3
docker rm etcd-1 etcd-2 etcd-3

THIS_IP=??
ETCD_HOSTS=s1=http://$THIS_IP:?,s2=http://$THIS_IP:?,s3=http://$THIS_IP:?
ETCD_TOKEN=??

CON_NAME=etcd-1
THIS_NAME=s1
THIS_CLIENT_PORT=?
THIS_PEER_PORT=?
docker run -d \
  --name $CON_NAME \
  --restart always \
  --net host \
  quay.io/coreos/etcd:v3 \
  /usr/local/bin/etcd \
  --name ${THIS_NAME} \
  --data-dir /etcd-data \
  --listen-client-urls http://${THIS_IP}:${THIS_CLIENT_PORT} \
  --advertise-client-urls http://${THIS_IP}:${THIS_CLIENT_PORT} \
  --listen-peer-urls http://${THIS_IP}:${THIS_PEER_PORT} \
  --initial-advertise-peer-urls http://${THIS_IP}:${THIS_PEER_PORT} \
  --initial-cluster $ETCD_HOSTS \
  --initial-cluster-state new \
  --initial-cluster-token ${ETCD_TOKEN}

CON_NAME=etcd-2
THIS_NAME=s2
THIS_CLIENT_PORT=?
THIS_PEER_PORT=?
docker run -d \
  --name $CON_NAME \
  --restart always \
  --net host \
  quay.io/coreos/etcd:v3 \
  /usr/local/bin/etcd \
  --name ${THIS_NAME} \
  --data-dir /etcd-data \
  --listen-client-urls http://${THIS_IP}:${THIS_CLIENT_PORT} \
  --advertise-client-urls http://${THIS_IP}:${THIS_CLIENT_PORT} \
  --listen-peer-urls http://${THIS_IP}:${THIS_PEER_PORT} \
  --initial-advertise-peer-urls http://${THIS_IP}:${THIS_PEER_PORT} \
  --initial-cluster $ETCD_HOSTS \
  --initial-cluster-state new \
  --initial-cluster-token ${ETCD_TOKEN}

CON_NAME=etcd-3
THIS_NAME=s3
THIS_CLIENT_PORT=?
THIS_PEER_PORT=?
docker run -d \
  --name $CON_NAME \
  --restart always \
  --net host \
  quay.io/coreos/etcd:v3 \
  /usr/local/bin/etcd \
  --name ${THIS_NAME} \
  --data-dir /etcd-data \
  --listen-client-urls http://${THIS_IP}:${THIS_CLIENT_PORT} \
  --advertise-client-urls http://${THIS_IP}:${THIS_CLIENT_PORT} \
  --listen-peer-urls http://${THIS_IP}:${THIS_PEER_PORT} \
  --initial-advertise-peer-urls http://${THIS_IP}:${THIS_PEER_PORT} \
  --initial-cluster $ETCD_HOSTS \
  --initial-cluster-state new \
  --initial-cluster-token ${ETCD_TOKEN}
```

