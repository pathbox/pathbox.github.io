---
layout: post
title: 最近工作总结(二十一)
date:   2018-11-20 11:50:06
categories: Work
image: /assets/images/post.jpg
---

### Base64对字符串或数据编码,得到的结果可能大小增大

### PDF 表单域填充中文不显示问题

PDF表单域，如果是Word转成PDF文件自带的表单域，当用代码进行表单域填充值时，很有可能不支持中文，只能英文字符才能被填充并显示。将原来自带的表单域，比如文本输入框删除，重新新建过表单域文本输入框，这样就解决了这个问题

### PDF 表单域只能一次填充

PDF的文字域填充只能填一次，比如我有六个文字域，第一次填充填了3个，第二次用第一次返回的PDF文件想要填剩下3个，发现不行了，表单域会失效

### e签宝签署接口

e签宝的接口，如果没有传accountId的，比如填充文字域接口，说明没有用到证书进行签名，只是操作PDF文件。有传accountId的，说明会用到证书对PDF文件进行签名，这接口操作是具有法律效力的

### MySQL ON DUPLICATE KEY UPDATE 对自增id的影响

MySQL主键自增有个参数 innodb_autonic_lock_mode,有三种值 0，1，2

0:自增时加表锁
1:不会锁表，建议值
2:不加锁，会有安全问题

REPLACE INTO ...每次插入的时候如果唯一索引对应的数据已存在，会删除原数据，然后重新插入新的数据，会导致id增大，但实际希望的情况是更新原有那条记录数据

INSERT INTO ...  ON DUPLICATE KEY UPDATE... 对主键id的影响

插入影响行数为1，更新影响行数为2

解决方案：

- 将ON DUPLICATE KEY UPDATE拆成两条SQL查询
- 不使用自增id主键

### tar 简记

#压缩

```
tar -czvf ***.tar.gz
tar -cjvf ***.tar.bz2
#解压缩
tar -xzvf ***.tar.gz
tar -xjvf ***.tar.bz2

参数：

-c  ：建立一个压缩档案的参数指令(create 的意思)；

-x  ：解开一个压缩档案的参数指令！

-t  ：查看 tarfile 里面的档案！

特别注意，在参数的下达中， c/x/t 仅能存在一个！不可同时存在！

因为不可能同时压缩与解压缩。

-z  ：是否同时具有 gzip 的属性？亦即是否需要用 gzip 压缩？

-j  ：是否同时具有 bzip2 的属性？亦即是否需要用 bzip2 压缩？

-v  ：压缩的过程中显示档案！这个常用，但不建议用在背景执行过程！

-f  ：使用档名，请留意，在 f 之后要立即接档名喔！不要再加参数！
```

### jpg文件和png文件大小比较
在像素width、height差不多的情况下，jpg文件大小比png小很多很多，不过清晰度也差不少

jpg是压缩较大并有损失失真的，png更好的保留了像素

### goto 在golang中的使用限制
首先不推荐使用

goto是不允许的，因为标签L跳过了变量v等声明和赋值，如果后面的代码访问v会有问题

### mailgu 免费版每小时只能发100封邮件

### Golang发邮件解决subject乱码显示问题

Golang发邮件遇到subject乱码显示问题，解决方法是需要对subject进行UTF-8编码，下面是代码示例:

```go
func SendToEvernote(user, password, host, to, subject,  body string) error {
    b64 := base64.NewEncoding("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/")
    from := mail.Address{user, user}
    toMail := mail.Address{to, to}
    header := make(map[string]string)
    header["From"] = from.String()
    header["To"] = toMail.String()
    header["Subject"] = fmt.Sprintf("=?UTF-8?B?%s?=", b64.EncodeToString([]byte(subject))) // 这一步，用base64对subject进行编码,然后指定subject为UTF-8编码
    header["MIME-Version"] = "1.0"
    header["Content-Type"] = "text/html; charset=UTF-8"
    header["Content-Transfer-Encoding"] = "base64"
    message := ""
    for k, v := range header {
        message += fmt.Sprintf("%s: %s\r\n", k, v)
    }
    message += "\r\n" + b64.EncodeToString([]byte(body))
    auth := smtp.PlainAuth("", user, password, host)
    err := smtp.SendMail(
        host+":25",
        auth,
        user,
        []string{toMail.Address},
        []byte(message),
    )
    if err != nil {
        panic(err)
    }
    return err
}
```

### 不要在home主页做太多复杂的逻辑或不稳定的逻辑

不要在home主页做太多复杂的逻辑或不稳定的逻辑，主页是主入口，需要保证100%的可用性

### 邮箱注册应该需要邮箱验证，手机号注册需要验证手机号，以防止恶意注册攻击
邮箱注册应该需要邮箱验证，以防止恶意注册攻击；同理手机号注册需要手机验证码验证，防止恶意注册攻击。
要不写一个脚本，不断的调用注册接口，生成的用户都是垃圾无效用户，还会把数据库给爆了。就变成了恶意的数据库攻击了

### PUT 是幂等的，而 PATCH 不是幂等的

PATCH是局部更新，PUT是所有字段更新

### Golang image 包处理图片注意点

```go
file, _ := os.Open(imagePath)
defer file.Close()

img, _, err := image.Decode(file)
if err != nil {
	return "", err
}
```
在不知道file图片格式的时候，统一使用image包的Decode，这要求默认导入图片格式对应的包
```go
"image"
_ "image/gif"
_ "image/jpeg"
"image/png"
```
比如这样，file就可以是gif、jpeg、jpg、png这几种格式的图片了

### rm -rf .git
将某个包copy到vendor目录下的时候，要将 `.git`这个目录删除，否则会导致无法将包正确git push

### 用户的新注册与注销
用户新注册生成，其所有相关信息尽量只保存到一张表，这样，要删除这个用户只要删这个表的记录即可。
用户的注销操作，如果某张表的记录删除了，就能注销这个用户，其他表的已存在的数据不影响其他用户，这样是最好的。

例子：一个用户新注册了，没有通过实名认证或购买记录，没有进行别的操作，现在只有一条有效信息在数据库表中，突然要求注销这个用户，重新注册，则把这条记录删除即可
