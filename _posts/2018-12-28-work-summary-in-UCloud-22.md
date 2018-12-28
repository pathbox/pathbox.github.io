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
