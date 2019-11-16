---
title: kebernetes 安装
categories: [devops]
tags: [运维]
date: 2019-11-16 17:58:26
---
# 结构
master:
```text
etcd
api-server
controllor-manager
scheduler
```
node
```text
kubelet
kube-proxy
docker
```
# 优化及常用工具安装
```bash
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
vi /etc/selinux/config

vi /etc/ssh/ssh_config
systemctl restart ssh

systemctl stop NetworkManager
systemctl disable NetworkManager

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum install -y bash-completion.noarch
yum install -y net-tools vim lrzsz wget tree screen lsof tcpdump

systemctl stop postfix
systemctl disable postfix
```
# master 安装
```bash
yum install -y etcd
vim /etc/etcd/etcd.conf

yum install kubernetes-master -y
vim /etc/kubernetes/apiserver

vim /etc/kubernetes/config
# 启动服务
systemctl start kube-apiserver.service
systemctl start kube-controller-manager.service
systemctl start kube-scheduler.service
# 查看状态
kubectl get componentstatus
```
# node 安装
```bash
yum install -y kubernetes-node

vim /etc/kubernetes/config

vim /etc/kubernetes/kubelet

systemctl start kubelet
systemctl enable kubelet

systemctl start kube-proxy
systemctl enable kube-proxy
```
# node 节点配置 flannel 网络插件
```bash
yum install -y flannel

vim /etc/sysconfig/flanneld

etcdctl set /atomic.io/network/config '{"Network":"172.17.0.0/16"}'

systemctl start flanneld
systemctl enable flanneld

systemctl restart docker
```
