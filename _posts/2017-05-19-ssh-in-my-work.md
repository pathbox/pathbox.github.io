---
layout: post
title: ssh in my work
date:   2017-05-19 10:51:06
categories: Work
image: /assets/images/post.jpg
---

我喜欢在`<hosts>`文件里给ip地址定义域名名称。比如一台测试服务器的ip地址是`<192.168.100.1>`

```
/etc/hosts

192.168.100.1 xxx.test.t1
```
然后，在.ssh/config中进行进一步配置

```
~/.ssh/config

host t1
 HostName 192.168.100.1
 User my_user
```

这样，在你能通过ssh正常登入t1这台服务器，就不必输入完整的登入命令，而只要

```
ssh t1
```

就可以`<ssh>`登入到t1这台服务器了

##### 给config增加料

我会在`<.ssh/config>`文件上增加这样的配置

```
ForwardAgent yes
ServerAliveInterval 60
```

`<ForwardAgent yes>` 当你使用跳板机去登入另一台服务器的时候，就能成功登入，否则会得到没有权限登入的错误提示。
比如：你先登入A机器，再从A机器登入到B机器。`<ForwardAgent>` 没有配置为yes，你就会登入失败，提示你没有权限。

如果你连接的是无限路由器，有时候会发现在ssh到一个远程主机以后过一段时间不操作就断线了。
ServerAliveInterval 60 能帮你解决这个问题。这里我设置的是60s的时间


##### 这个操作能简单检查你的ssh 是否是正常运行
```
ssh -T git@github.com
# Attempt to SSH in to github
Hi username! You've successfully authenticated, but GitHub does not provide
shell access.
```

##### You can check that your key is visible to `<ssh-agent>` by running the following command:

```
ssh-add -L
ssh-add -l
```

##### If the command says that no identity is available, you'll need to add your key:

```
ssh-add  ~/.ssh/id_rsa
```

```
On Mac OS X, ssh-agent will "forget" this key, once it gets restarted during reboots. But you can import your SSH keys into Keychain using this command:


/usr/bin/ssh-add -K yourkey
```

如果你用`<zsh>`的话，可以在`<.zshrc>`中加入这一句

```
ssh-add  ~/.ssh/id_rsa
```

参考连接： https://developer.github.com/guides/using-ssh-agent-forwarding/
