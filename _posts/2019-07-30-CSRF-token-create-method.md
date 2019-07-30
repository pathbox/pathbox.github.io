---
layout: post
title: CSRF Token 生成方法原理
date:  2019-07-30 16:50:06
categories: Work
image: /assets/images/post.jpg
---

最近又看了一下CSRF攻击的内容，将CSRF token的生成算法也熟悉了一下，在此做总结。

对CSRF攻击的防御措施主要有两种

- 检查HTTP Header Referer字段: 通常来说，Referer字段应和请求的地址位于同一域名下
- 添加校验token

生成CSRF token的具体逻辑，下面使用的是Rails中的生成方法做例子，我看了`gorilla/csrf`也是类似的方法

```ruby
# 首先生成真实的raw_token并存到session或cookie中
def real_csrf_token(session) # :doc:
  session[:_csrf_token] ||= SecureRandom.base64(AUTHENTICITY_TOKEN_LENGTH)
  Base64.strict_decode64(session[:_csrf_token])
end

# 看到raw_token 只是一个已知长度的随机字符串，然后再base64之后，存到session中
raw_token = real_csrf_token(session)

# 构造给表单hidden或ajax的mask_token

def mask_token
  one_time_pad = SecureRandom.random_bytes(AUTHENTICITY_TOKEN_LENGTH)
  encrypted_csrf_token = xor_byte_strings(one_time_pad, raw_token) # xor计算
  masked_token = one_time_pad + encrypted_csrf_token
	Base64.strict_encode64(masked_token)
end

# mask_token在前端页面是可以看到的

# 如果进行对比校验?
# 将前端传过来的masked_token 进行unmask 得到一个 unmask_token 再和session中的session[:_csrf_token]进行比较，相等说明校验通过,unmask 的过程就是mask的逆向过程

def unmask_token(masked_token) # :doc:
  one_time_pad = masked_token[0...AUTHENTICITY_TOKEN_LENGTH]
  encrypted_csrf_token = masked_token[AUTHENTICITY_TOKEN_LENGTH..-1]
  xor_byte_strings(one_time_pad, encrypted_csrf_token)
end

unmask_token == session[:_csrf_token] ? Yes : No
```

更详细的理解:

- 在这一次session的生命周期中，`session[:_csrf_token]` 是固定的
- 每次给前端的mask_token是不一样的，是因为每次的`one_time_pad`是不一样的
- 但是，每次将mask_token unmask逆向之后得到的real_token应该是一样的，然后和`session[:_csrf_token]`比较即可

所以，如果攻击者要破解CSRF token，他需要能够在session这次生命周期内拿到`session[:_csrf_token]`
如果`session[:_csrf_token]`存在cookie中，攻击者需要破解cookie得到真实数据，可以设置cookie的httponly属性为true，这样cookie就只能通过服务器读取不能通过javascript读取
