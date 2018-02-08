---
layout: post
title: after_commit vs after_save
date:   2016-05-15 15:30:06
categories: Rails
tag: rails
image: /assets/images/post.jpg
---



在rails中使用after_ 或before_ 的回调方法，能方便的帮我们处理代码逻辑。
after_save方法：顾名思义就是在save 之后进行操作。
rails guide中的一句：[after_save runs both on create and update, but always after the more specific callbacks after_create and after_update, no matter the order in which the macro calls were executed.](http://guides.rubyonrails.org/active_record_callbacks.html)

```ruby

  class User < ActiveRecord::Base
    after_save :notify_email, on: [:update] :if => Proc.new {|record| record.previous_changes.key?(:password) && record.previous_changes[:password].first != record.previous_changes[:password].last}
    #after_commit :notify_email, on: [:update] :if => Proc.new {|record| record.previous_changes.key?(:password) && record.previous_changes[:password].first != record.previous_changes[:password].last}
    #修改密码后邮件通知用户
    def notify_email
        # ...
    end
  end


```

after_save(after_commit)可以使用 on 来指定是update还是create操作触发回调,如果没有指定,这两种操作都会触发回调
下面代码的例子转自：http://jakeyesbeck.com/2016/03/27/five-more-active-record-features-you-should-be-using/

To add a new Book to a queue for a Review after it is saved:

```ruby

class Book < ActiveRecord::Base
  after_save :enqueue_for_review

  def enqueue_for_review
    ReviewQueue.add(book_id: self.id)
    Logger.info("Added #{self.id} to ReviewQueue")
  end
end

```

We can assume that the ReviewQueue is a key/value storage (Redis or something similar) backed object whose purpose is to place new Books into a queue for critics to review. The Logger class simply outputs text to stdout.

When everything goes well, the callback works beautifully:

```ruby

Book.create!(
  title: 'A New Book',
  author_id: 3,
  content: 'Blah blah...'
)
#=> Added 4 to ReviewQueue
#=> <Book id: 4, title: 'A New Book' ..>

ReviewQueue.size
#=> 1

```
However, if this code was wrapped in a transaction, and that transaction fails,
the Book will not persist but the element on in the Redis-backed ReviewQueue remains.

```ruby

ActiveRecord::Base.transaction do
  Book.create!(
    title: 'A New Book',
    author_id: 3,
    content: 'Blah blah...'
  )

  raise StandardError, 'Something Happened'
end

#=> Added 4 to ReviewQueue
#=> Error 'Something Happened'

ReviewQueue.size
#=> 1
Book.find_by(title: "A New Book")
#=> nil

```
This code has now generated an element on the ReviewQueue for a non-existent Book; however,
if the after_commit callback is used, this problem will fade away.
Replacing after_save with after_commit:

```ruby

class Book < ActiveRecord::Base
  after_commit :enqueue_for_review

  # ...
end

ActiveRecord::Base.transaction do
  Book.create!(
    title: 'A New Book',
    author_id: 3,
    content: 'Blah blah...'
  )

  raise StandardError, 'Something Happened'
end

#=> Error 'Something Happened'

ReviewQueue.size
#=> 0

```

Awesome, the after_commit callback is only triggered after the record is persisted to the database, exactly what we wanted.

中文解释: 在这个例子中，当触发回调的create操作外层包裹了transaction时候，使用after_save操作，如果transaction有异常保存失败，
即使book对象先是create成功了(即save操作完成过一次)，但是由于后面的代码报错，触发了transaction的rollback，使得create操作撤销。
使得book对象没有create一个新的对象，这是正确的。但是，after_save操作由于book有save操作完成过一次，在save操作完成时就触发了回调
(那时候raise代码还没执行到,rollback操作还没完全执行。可以认为after_save在save操作后触发回调极快，比遇到异常触发rollback回滚还快，使得after_save的回调方法被执行了，但其实新建的book对象由于异常又回滚了)。
这样ReviewQueue就新建了一个
对象。然而，ReviewQueue对象所对应的book_id(book对象)已经由于transaction的异常而rollback回滚撤销保存了，数据库里是没有这个book对象。
也就使得ReviewQueue 的book_id 在Book中是不存在的。如果有相关的操作就会报根据这个book_id 找不到book对象
而用 after_commit 操作就解决这个问题了。after_commit 操作会监听最外层的transaction范围内都正常执行完，才会触发回调。如果在最外层的
transaction范围内有异常发生，都会发生回滚。而after_save只会监听create这个操作范围内的代码异常(这个例子中可以认为after_commit监听的异常代码范围大于after_save)
总结: 如果方法外层有transaction包裹进行的需要触发的回调，请务必使用after_commit。这样保证after_commit监听的异常代码范围最大

你还可以看另一个例子: [Use after_commit instead of active record callbacks to avoid unexpected errors](http://codebeerstartups.com/2012/11/use-after_commit-instead-of-active-record-callbacks-to-avoid-unexpected-errors/)
这篇文章大概的意思是:

1. main process
2. worker process
3. BEGIN

after_create:
INSERT comment in comments table
# return id 10 for newly-created notification
SELECT * FROM notifications WHERE id = 10
COMMIT

after_commit:
INSERT comment into comments_table
return id 10 for newly-created notification
COMMIT
SELECT * FROM notifications WHERE id = 10

> You won’t see any issue in development, as local db can commit fast. But in production server, db traffic might be huge, worker probably finish faster than transaction commit. e.g

> In this case, the worker process query the newly-created notification before main process commits the transaction,it will raise NotFoundError, because transaction in worker process can’t read uncommitted notification from transaction in main process.


而用after_commit 操作就不会了, 子进程(asyns_send_notification)一定会等整个代码进行到commit操作之后，才会触发。从这里也可以看出after_commit监听的范围比after_save要大
