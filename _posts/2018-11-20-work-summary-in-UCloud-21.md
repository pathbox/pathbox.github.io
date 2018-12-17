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
