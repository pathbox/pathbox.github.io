---
layout: post
title: 最近工作总结(47)
date:  2021-03-03 20:00:00
categories: Work
image: /assets/images/post.jpg


---

​    

### etcd集群启动方式

```

$ etcd -name infra0 -initial-advertise-peer-urls http://10.0.1.10:2380 \
 -listen-peer-urls http://10.0.1.10:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new

$ etcd -name infra1 -initial-advertise-peer-urls http://10.0.1.11:2380 \
 -listen-peer-urls http://10.0.1.11:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new

$ etcd -name infra2 -initial-advertise-peer-urls http://10.0.1.12:2380 \
 -listen-peer-urls http://10.0.1.12:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new

```

