---
layout: post
title: 记一次Go websocket　项目内存泄露排查
date:   2017-05-26 20:20:06
categories: Go
image: /assets/images/post.jpg
---

655.15MB 17.91% 17.91%   655.15MB 17.91%  runtime.rawstringtmp
  643.69MB 17.59% 35.50%   693.39MB 18.95%  runtime.mapassign
  561.81MB 15.36% 50.86%   561.81MB 15.36%  runtime.makemap
  324.61MB  8.87% 59.73%   324.61MB  8.87%  gopkg.in/mgo%2ev2.copySession
  200.96MB  5.49% 65.22%   200.96MB  5.49%  github.com/gorilla/websocket.newConn
  195.02MB  5.33% 70.55%   195.52MB  5.34%  context.WithCancel
  167.04MB  4.57% 75.12%  1397.52MB 38.20%  net/http.readRequest
   92.51MB  2.53% 77.65%   918.37MB 25.10%  net/textproto.(*Reader).ReadMIMEHeader
   89.01MB  2.43% 80.08%    89.01MB  2.43%  github.com/gorilla/securecookie.New
   86.51MB  2.36% 82.45%   313.05MB  8.56%  github.com/kidstuff/mongostore.(*MongoStore).New

https://stackoverflow.com/questions/39404625/golang-websocket-memory-leak

http://www.gorillatoolkit.org/pkg/sessions
