---
layout: post
title: 最近工作总结(45)
date:  2021-01-05 20:00:00
categories: Work
image: /assets/images/post.jpg


---

 

### Docker 服务异常导致所有容器失效

由于主机服务突然断电或重启，导致Docker服务所有容器缺失了state.json文件。`docker info` `docker ps`等命`令会卡住，无法继续执行。到/var/log/docker 中查看输出日志,得到: 

```sh
evel=fatal msg="open /var/run/docker/libcontainerd/containerd/054f92393f757e0418b014ed1fa35673fbce2293de43e42153f4e10ec4910c77/state.json: no such file or directory
```

 解决方式:

```
1.service docker stop
2.Go to /var/run/docker and delete any directory related to the container id
3.Go to /var/lib/docker and delete any directory related to the container id
4.service docker start
```

有可能需要将目录下的所有 `container id`的文件目录都删了，因为他们都失效了。

docker 重启后，会恢复正常。但是容器都没有了，需要重新创建过。

如果是有CI/CD的，重新发布过服务即可，如果没有，那么所有失效容器重新创建会是比较繁琐了过程

