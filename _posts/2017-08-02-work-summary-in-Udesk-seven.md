---
layout: post
title: 最近工作总结(七)
date:   2017-08-02 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### 使用正则快速获取括号内的字符串
result = file_name.match(/\A.*\((\S+)\).*\Z/)
result1 = file_name.match(/\A.*\[(\S+)\].*\Z/)
 p $1
 p result.to_a

 ##### 构建了一个sidekiq worker的思考
 当你新建了一个sidekiq worker，需要思考这个sidekiq woker 是否在你的使用环境需要真的被新建，
 是否有条件满足的时候才需要新建这个worker，比如某个开关，或某个实例对象存在。

 当新建出一个sidekiq worker，思考其中的代码逻辑，是否不再满足某种条件的时候，就直接return，而不再让代码继续执行。

 思考上面的两点，可以减少worker的新建次数，可以然后worker真正有效的执行，减少不必要的执行次数
