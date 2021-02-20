---
layout: post
title: 最近工作总结(46)
date:  2021-02-05 20:00:00
categories: Work
image: /assets/images/post.jpg


---

   

### Kong+go plugin server 对上传文件接口处理的bug

kong的网关接口出现了内存一直上升不释放，导致Pod配置的内存被耗尽的情况

![WeChatWorkScreenshot_ba0fa94b-85a6-458f-928a-d9afb34a53e3](/Users/pathbox/Desktop/WXWork Files/Image/2021-02/WeChatWorkScreenshot_ba0fa94b-85a6-458f-928a-d9afb34a53e3.png)

服务的日志中打印了大量该日志，从日志上看是mmap的读写操作。

```lua
while not ngx.worker.exiting() do 
  kong.log.notice("Starting"..server_def.name or "")
  server_def.proc = assert(ngx_pipe.spawn(server_def.start_command, {
        merge_stderr = true
      }))
end

while not ngx.worker.exiting() do 
  kong.log.notice("Starting"..server_def.name or "")
  server_def.proc = assert(ngx_pipe.spawn(server_def.start_command, {
        merge_stderr = true,
        buffer_size = 40960 
      }))
end
```

解决方法：buffer_size默认是4096byte，这里将其重置扩大了10倍

openresty这里buffer_size用的默认值。 导致读取go plugin server返回的内容时，由于上传的文件可能是几M，会不断尝试申请更大的内存，直到申请到足够大的内存。 但是由于lua gc的释放内存逻辑，之前申请的内存也不会及时释放，导致短时间内存上升，将Pod的内存耗尽

### 数据库多地，缓存非多地导致的查询问题

A数据库会同步到B数据库，但写操作只操作A数据库。且在A区域有一个缓存集群，所有数据库都共用该缓存集群。

问题流程: 一个写操作 => 将缓存删除 => A的数据还未同步到B数据库 => B地域有读请求,从B数据库读取到了旧的数据 => 此时没有缓存，则B的读操作会更新缓存 => 旧的数据又更新为了缓存 => A的数据同步到了B数据库,但是B数据库不会删除缓存，将旧的缓存数据库删除

快速的解决方案: A数据同步到B数据库的时候也将对应的缓存删除。但这样其实B地区的用户第一个请求时候，还是可能读取到的是B数据库的旧数据。此方案并非完全一致性，是最终一致性，有实时性问题。



### Redis超时大于网关接口超时而导致的诡异情况

redis缓存操作超时，该操作并非异步处理，而超时时间达到了2分钟。超时报错之后程序会继续执行，但实际网关的超时时间是1分钟，已经超时返回给了前端。所以出现了，从日志上看后端逻辑都执行了，只是中间延迟了2分钟，而前端操作接受到了接口超时的返回而没有继续进行业务下一步的接口调用

