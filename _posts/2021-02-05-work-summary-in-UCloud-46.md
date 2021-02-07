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