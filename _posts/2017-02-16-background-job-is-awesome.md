---
layout: post
title: Background job is awesome for your application
date: 2017-02-16 17:00:06
categories: Thought
image: /assets/images/post.jpg
---

What is the background job? It is simple that is running in the background.

It is like a worker that running in the background and don't impact performance in the font end.

Background job has three factors:

1. asynchronous job

2. queue

3. callback

##### asynchronous job
It doesn't block the performance in the font end. You don't need to wait for the job, and you can do other things that
you want. Just after some minutes, you come back to check that if it is done, and get the result.

##### queue
We need a queue that stores the background jobs. (You can choose a stack, too)

It is a nice approach. The background job is processed in order.

Of course, you can design a complex queue that it contains many conditions and weights.

Sometimes simple queue is enough.

###### callback
Callback is not necessary for background job.
You can send the notification of the background job's result to the font end.
Maybe you don't do it. Because we don't need get the result right now.
Or we don't need do other things　after the background job.
Be patient. You can get the surprise that yeah, I get the result form the background job.


##### Why do we need the background job?
The only reason is: We want to build a high and friendly performance application.

Background job can help us do it.

##### When do we make a background job?

```
I make a list:

Send a notification message

Send a email

Send a lot of notification messages

Send a lot of emails

Call the third part's api to get some info results (then insert the results to Databases)

Bulk operations. For example, Bulk write operations for MySQL

Create a record that the font-end don't need it from response. For example some logs

Statistics operations that need a long time

Calculation operations that need a long time

Analysis operations that need a long time

Import operations that need a long time

Export operations that need a long time

Synchronization operations that need a long time

Micro-Services operations that need a long time

Upload operations that need a long time

Download operations that need a long time and don't want to wait in font-end

Webhook operations that need a long time

Operations that need a long time HaHahhhhhhhhhhhhhhhhhh

You don't need get the response for the font-end
```

##### But don't fuck the background job, it is not almighty.
You should do enough logs work in your background jobs logic code.

That you can locate your problem when the errors happened and job fails quickly.
Or it is a big hole that you can't get the reasons. You can't control the background jobs.
Don't make them become Baby Bear!

##### Make a nice background job
Background job is really nice for our application.

Control it and take care of it, it can make a lot of benefits.
