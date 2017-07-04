---
layout: post
title: 最近工作总结(六)
date:   2017-07-03 20:16:06
categories: Work
image: /assets/images/post.jpg
---

##### 如果调用第三方接口，请别忘了超时机制
周末的时候，线上出了问题。是由于IPIP服务商的服务器出了问题，我们调的接口迟迟没有返回值。
导致我们项目中，开了很多goroutine，每个goroutine中会请求IPIP的接口分析ip地址，由于IPIP的服务器没能及时相应，
导致大量的goroutine都阻塞在了那里。所以， 不要“信任”第三方服务，调用他们服务接口的时候，应该使用超时机制。

优化方案：使用了本地化IP地址的方案，使用了这个

> https://github.com/lionsoul2014/ip2region

减少了调用IPIP的次数。这样， 内存也一下少了200+M。 Perfect!
