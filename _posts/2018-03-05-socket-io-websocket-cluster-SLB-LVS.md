---
layout: post
title:  Socket.io 和WebSocket 集群方案总结
date:   2018-03-05 20:00:06
categories: Server
image: /assets/images/post.jpg
---

Socket.io 大概可以分为两种长连接的模式: polling 和 websocket. polling方式可以用在不支持websocket的浏览器中,比如IE7-IE9.

Socket.io Client 在向服务端发起连接请求的时候,会检查浏览器是否支持websocket,如果支持,会使用websocket进行连接,如果不支持,会使用polling方式. 你也可以指定使用哪种方式:

```js
socket = io(serverUrl, {transports: ['websocket'], upgrade: false});
```

Socket.io 对于WebSocket的优势之一就是能为不支持WebSocket的浏览器也提供一种"长连接"方案.

而在集群方案中,个人觉得,WebSocket 比 Socket.io 优势很多,请看下面的内容:

### SLB(LVS-WebSocket) + WebSocket Cluster

阿里的SLB(LVS)负载均衡支持了WebSocket.如果要支持 https,wss, 可以把证书放在SLB上.

示意:
```
SLB(LVS) => WebSocket Cluster
```

这种方案能支持的socket连接数,应该不受IP端口65535数量的限制,而是由服务器的资源,网络带宽等决定.

个人认为这是高性能高并发连接数量的WebSocket集群,最好的集群方案.

### Nginx + WebSocket Cluster

Nginx 也支持了WebSocket的负载均衡代理.

示意:
```
Nginx => WebSocket Cluster
```

Nginx性能比LVS低一些, Nginx负载均衡代理受IP端口65535限制. 也就是理论上,集群最多支持65535个socket 连接.

### SLB(LVS-WebSocket) + Socket.io Cluster

SLB的WebSocket负载均衡不支持Socket.io Cluster. 使用的负载均衡算法应该是轮询模式. Socket.io 建立连接的时候和WebSocket不同. Socket.io 发起握手和建立连接这两个步不在同一个tcp连接中完成. 这样就会造成,发起握手的请求发到了A机器,建立连接的请求发到了B机器. B机器发现之前没有收到过握手请求,于是会断开连接,导致连接失败. 在Chrome上,用go-socket.io 做后端, 查看日志是 连接连上了,又立马断开了. 查看Chrome Network, 发现报了invalid sid 的错误.

也有人说使用LVS的source hash, 解决Socket.io Cluster 的问题.

### SLB(LVS-TCP) + Nginx + Socket.io Cluster

SLB TCP 负载均衡可以支持Socket.io Cluster. 连接能够成功建立. 但是,由于SLB 使用的是TCP负载均衡,无法在SLB上设置SSL证书,第二是,如果SLB前面加高防,由于SLB TCP是四层代理, Socket.io 后端无法获得真实的client IP, 获得的是高防的IP.

如果要使用https, 可以在socket.io 的前面挂一个Nginx, 在这个Nginx上配https. 这样又出现了Nginx连接数瓶颈的问题.而且,如果要http重定向为https,极端情况,所有请求都是http,http在Nginx转为https的过程中,也是要占用端口数的.所以建议同时支持http和https，而不把http强制重定向为https.

不过这样和单独使用Nginx, 理论上能支持的并发连接数为: 65535*N(集群服务器数量)

### Nginx(ip_hash) + Socket.io Cluster

这是官方提供的解决方案.缺点很明显: "65535"连接数的限制.不过,对于中小型的网站服务,应该是完全够用的.

```
upstream io_nodes {
  ip_hash;
  server 127.0.0.1:6001;
  server 127.0.0.1:6002;
  server 127.0.0.1:6003;
  server 127.0.0.1:6004;
}

server {
  listen 3000;
  server_name io.yourhost.com;
  location / {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_pass http://io_nodes;
  }
}
```

而且,Nginx是七层代理, 设置 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`, 即使最前面有高防,也可以拿到client IP.

对于OpenResty 的 `lua-resty-websocket`, 性能和连接数是否比一般Nginx强,这个有待尝试

总结

1. 想要支持尽可能多的WebSocket连接, 选择 `SLB(LVS-WebSocket) + WebSocket Cluster`

2. 必须使用Socket.io, 选择 `SLB(LVS-TCP) + Nginx + Socket.io Cluster`

3. Socket.io 想要获得Client IP, 使用 `Nginx(ip_hash) + Socket.io Cluster`

4. 如果使用Nginx,不要将http重定向为https,可以让两者同时支持.

5. go-socket.io 现有业务,1w连接占2.5-2.8G内存

6. 其他 七层负载均衡：负载均衡器与客户端及后端的服务器会分别建立一个TCP连接。即两次TCP连接
