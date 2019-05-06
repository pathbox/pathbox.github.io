---
layout: post
title: 最近工作总结(27)
date:  2019-05-06 17:00:06
categories: Work
image: /assets/images/post.jpg
---

### ASFK-理解cookie和session
- Cookie由浏览器管理
- Cookie是明文的，用于存储非敏感机密数据
- Cookie具有不可跨域名性
- Cookie中使用Unicode字符时需要对Unicode字符(如中文)进行编码，否则会乱码
- Cookie不仅可以使用ASCII字符与Unicode字符，还可以使用二进制数据。例如在Cookie中使用数字证书，提供安全度。使用二进制数据时也需要进行编码。注意：本程序仅用于展示Cookie中可以存储二进制内容，并不实用。由于浏览器每次请求服务器都会携带Cookie，因此Cookie内容不宜过多，否则影响速度。Cookie的内容应该少而精
- Cookie的maxAge决定着Cookie的有效期，单位为秒（Second）。Cookie中通过getMaxAge()方法与setMaxAge(int maxAge)方法来读写maxAge属性
- Cookie并不提供修改、删除操作。如果要修改某个Cookie，只需要新建一个同名的Cookie，添加到response中覆盖原来的Cookie。如果要删除某个Cookie，只需要新建一个同名的Cookie，并将maxAge设置为0，并添加到response中覆盖原来的Cookie。注意是0而不是负数。负数代表其他的意义。
- 正常情况下，同一个一级域名下的两个二级域名如www.helloweenvsfei.com和 images.helloweenvsfei.com也不能交互使用Cookie，因为二者的域名并不严格相同。如果想所有 helloweenvsfei.com名下的二级域名都可以使用该Cookie，需要设置Cookie的domain参数，例如：
```
Cookie cookie = new Cookie("time","20080808"); // 新建Cookie

cookie.setDomain(".helloweenvsfei.com");           // 设置域名

cookie.setPath("/");                              // 设置路径

cookie.setMaxAge(Integer.MAX_VALUE);               // 设置有效期

response.addCookie(cookie);                       // 输出到客户端
```

- Cookie的路径
```
domain属性决定运行访问Cookie的域名，而path属性决定允许访问Cookie的路径（ContextPath）。例如，如果只允许/sessionWeb/下的程序使用Cookie，可以这么写：

Cookie cookie = new Cookie("time","20080808");     // 新建Cookie

cookie.setPath("/session/");                          // 设置路径

response.addCookie(cookie);                           // 输出到客户端

设置为“/”时允许所有路径使用Cookie。path属性需要使用符号“/”结尾。name相同但domain相同的两个Cookie也是两个不同的Cookie。

注意：页面只能获取它属于的Path的Cookie。例如/session/test/a.jsp不能获取到路径为/session/abc/的Cookie。使用时一定要注意。
```
- Cookie的安全属性
```
HTTP协议不仅是无状态的，而且是不安全的。使用HTTP协议的数据不经过任何加密就直接在网络上传播，有被截获的可 能。使用HTTP协议传输很机密的内容是一种隐患。如果不希望Cookie在HTTP等非安全协议中传输，可以设置Cookie的secure属性为 true。浏览器只会在HTTPS和SSL等安全协议中传输此类Cookie。下面的代码设置secure属性为true：


Cookie cookie = new Cookie("time", "20080808"); // 新建Cookie

cookie.setSecure(true);                           // 设置安全属性

response.addCookie(cookie);                        // 输出到客户端

提示：secure属性并不能对Cookie内容加密，因而不能保证绝对的安全性。如果需要高安全性，需要在程序中对Cookie内容加密、解密，以防泄密。

```
- JavaScript操作Cookie

Cookie是保存在浏览器端的，因此浏览器具有操作Cookie的先决条件。浏览器可以使用脚本程序如 JavaScript或者VBScript等操作Cookie。这里以JavaScript为例介绍常用的Cookie操作。例如下面的代码会输出本页面 所有的Cookie。

<script>document.write(document.cookie);</script>

由于JavaScript能够任意地读写Cookie，有些好事者便想使用JavaScript程序去窥探用户在其他网 站的Cookie。不过这是徒劳的，W3C组织早就意识到JavaScript对Cookie的读写所带来的安全隐患并加以防备了，W3C标准的浏览器会 阻止JavaScript读写任何不属于自己网站的Cookie。换句话说，A网站的JavaScript程序读写B网站的Cookie不会有任何结果。

- Session往往以key-value的结构存储

- Session是另一种记录客户状态的机制，不同的是Cookie保存在客户端浏览器中，而Session保存在服务器上。客户端浏览器访问服务器的时候，服务器把客户端信息以某种形式记录在服务器上。这就是Session。客户端浏览器再次访问时只需要从该Session中查找该客户的状态就可以了。

如果说Cookie机制是通过检查客户身上的“通行证”来确定客户身份的话，那么Session机制就是通过检查服务器上的“客户明细表”来确认客户身份。Session相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了

Session保存在服务器端。为了获得更高的存取速度，服务器一般把Session放在内存里。每个用户都会有一个独立的Session。如果Session内容过于复杂，当大量客户访问服务器时可能会导致内存溢出。因此，Session里的信息应该尽量精简。

Session生成后，只要用户继续访问，服务器就会更新Session的最后访问时间，并维护该Session。用户每访问服务器一次，无论是否读写Session，服务器都认为该用户的Session“活跃（active）”了一次。

- 用redis存储session。redis非常适合存储session，满足了key-value存储，速度足够快，有过期时间。
- cookie和session的关系:
1. session数据可以存储在cookie中，为了避免安全信息暴露，需要加密存与cookie
2. session存于redis中，但需要一个Session ID(或session key)存于cookie，用于客户端唯一标记信息。请求时，cookie中带有Session ID 服务端能够取到，则可以在redis中根据Session ID 获取到具体的session数据用于逻辑操作。
3. 浏览器将cookie关了，则session功能也会失效，需要存于URL中
