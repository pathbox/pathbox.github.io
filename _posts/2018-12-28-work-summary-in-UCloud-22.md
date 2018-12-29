---
layout: post
title: 最近工作总结(二十二)
date:   2018-12-28 15:33:06
categories: Work
image: /assets/images/post.jpg
---

### docker使用小结

- docker search name 搜索你要的镜像
- docker pull 下载你要的镜像
- docker build -t my_tomcat_war/tomcat:v1 . 通过Dockerfile构建容器
- 想要删除image需要先停止并删除容器

```
docker stop container-id
docker rm container-id
docker rmi image-id
```

- docker run -v /tmp/my_war_logs:/usr/local/tomcat/logs -d -p 8080:8080 my_tomcat_war/tomcat:v1 # 第二个端口号为docker中Tomcat war包服务的端口号，需要和server.xml中的配置的端口号一致，默认是8080 挂载的日志目录，docker运行用户需要有读写和进入的权限

- docker exec -it container-id /bin/bash 可以进入到容器终端，进行命令交互。这时的容器终端是一个非常简单的shell环境，很多软件或命令都没有

- 容器拷贝到宿主主机 docker cp mycontainer：/opt/testnew/file.txt /opt/test/

- 宿主主机拷贝到容器 docker cp /opt/test/file.txt mycontainer：/opt/testnew/

- 不管容器有没有启动，拷贝命令都会生效

- yum install net-tools -y 安装`net-tools`后，可以ifconfig查看docker ip

- docker stop $(docker ps -a -q) stop所有容器

- 默认启动容器内服务 `docker run -idt container_id /bin/start-service.sh, ps:start-service.sh为镜像内的脚本`

- `ctop` 简单查看docker container的状态

### docker 列出每个容器的IP

常用方法有两种:
方法一:

docker inspect 容器ID | grep IPAddress

方法二:

```
查看docker name：
sudo docker inspect -f='{{.Name}}' $(sudo docker ps -a -q)
查看dockers ip：
sudo docker inspect -f='{{.NetworkSettings.IPAddress}}' $(sudo docker ps -a -q)
综上，我们可以写出以下脚本列出所有容器对应的名称，端口，及ip
docker inspect -f='{{.Name}} {{.NetworkSettings.IPAddress}} {{.HostConfig.PortBindings}}' $(docker ps -aq)
```

### 支付的时候要校验价格和付的钱是否一致
支付的时候要校验价格和付的钱是否一致，要不然可能会出现用实际支付了1元购买1万元产品的情况，这就是系统漏洞

### Cookie 的安全属性

##### Secure
>Cookie通信只限于加密传输，指示浏览器仅仅在通过安全/加密连接才能使用该Cookie。如果一个Web服务器从一个非安全连接里设置了一个带有secure属性的Cookie，当Cookie被发送到客户端时，它仍然能通过中间人攻击来拦截。

##### HttpOnly
>Cookie的HttpOnly属性，指示浏览器不要在除HTTP（和 HTTPS)请求之外暴露Cookie。一个有HttpOnly属性的Cookie，不能通过非HTTP方式来访问，例如通过调用JavaScript(例如，引用document.cookie），因此，不可能通过跨域脚本（一种非常普通的攻击技术）来偷走这种Cookie
