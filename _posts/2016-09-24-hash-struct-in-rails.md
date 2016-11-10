---
layout: post
title: Hash struct in rails
date:   2016-09-24 17:58:06
categories: rails
image: /assets/images/post.jpg
---

最近遇到了一个坑，关于rails中的hash struct。问题不难，但非常容易让人忽略。

##### Action Controller Parameters

```ruby
params = ActionController::Parameters.new({
  person: {
    name: 'Francesco',
    age:  22,
    role: 'admin'
  }
})

permitted = params.require(:person).permit(:name, :age)
permitted            # => <ActionController::Parameters {"name"=>"Francesco", "age"=>22} permitted: true>
permitted.permitted? # => true

Person.first.update!(permitted)
# => #<Person id: 1, name: "Francesco", age: 22, role: "user">

p params.class #=>  ActionController::Parameters

hash = Hash.new(
  {
    person: {
      name: 'Francesco',
      age:  22,
      role: 'admin'
    }
  }
)

p hash.class #=> Hash
```

继续

```ruby
params = ActionController::Parameters.new
params.permitted? # => false

ActionController::Parameters.permit_all_parameters = true

params = ActionController::Parameters.new
params.permitted? # => true

params = ActionController::Parameters.new(a: "123", b: "456")
params.permit(:c)
# => <ActionController::Parameters {} permitted: true>

ActionController::Parameters.action_on_unpermitted_parameters = :raise

params = ActionController::Parameters.new(a: "123", b: "456")
params.permit(:c)
# => ActionController::UnpermittedParameters: found unpermitted keys: a, b
params.permit(:a,:b)
#=> {"a"=>"123", "b"=>"456"}
```

##### ActionController::Parameters 和 Hash的最大区别

```ruby
params = ActionController::Parameters.new(key: 'value')
params[:key]  # => "value"
params["key"] # => "value"

hash = {a: 1, "b"=> 2}

hash[:a]    #  => 1
hash["a"]   #  => nil
hash[:b]    #  => nil
hash["b"]   #  => 2
```

##### 总结

ActionController::Parameters 不区分 [:a] ["a"] 而 Hash 区分。

a = ActionController::Parameters.new({a: 1})

a = ActionController::Parameters.new({"a"=> 1})

上面两个 a[:a] a["a"] 的结果都可以取到 1

##### 我的坑
字段field 用了 serialize :field, Hash
前端传值的时候是传的 ActionController::Parameters, 而更改字段值的时候有可能用的是Hash，导致有的记录 field是ActionController::Parameters，有的是hash都用 ["a"] 这样取值的话，就不会有错了

##### 补充 使用 HashWithIndifferentAccess

```ruby

a = {"key" => "keyword"}
b = HashWithIndifferentAccess.new(a)
b.class #=>  ActiveSupport::HashWithIndifferentAccess
b[:key] #=> "keyword"
b["key"] #=> "keyword"

```
