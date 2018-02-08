---
layout: post
title: 使用privoxy代理shadowsocks让Teminal命令行实现VPN
date:   2017-11-18 15:08:06
categories: Tool
image: /assets/images/post.jpg
---

最近买的云梯VPN挂了快两月了,现在还没好,也没发邮件通知等,提工单也不再有回复.

而在`Go`开发中,有一些包的导入是需要翻墙到谷歌源的.在运维小哥的协助下,使用privoxy解决了这个问题.

Ubuntu 14.04, 执行

```
sudo apt-get install privoxy
```

然后 sudo vim /etc/privoxy/config

在最后一行加上你的`shadowsocks`的本地代理配置

```
forward-socks5 / 127.0.0.1:1080 .
```

开启`shadowsocks`

其他的配置如果你看不懂或还不需要,就不用改动,使用默认的就可以

执行

```
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
```

或者把其加入到 .zsh_profile(我用的zsh)

之后，进行 `go get google.golang.org/xxx` 之类下载的谷歌源的代码包，就能成功了

Just mark
