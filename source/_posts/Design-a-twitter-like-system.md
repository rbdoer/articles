---
title: Design a twitter like system
date: 2017-09-11 14:09:17
tags:
categories: 系统设计
---

## 需求分析

### 需要考虑的问题

- 数据存储：用户关系存储，feed存储，转发评论存储
- feed流的实现：拉取？还是推送？
- 关注了很多人的用户，拉取耗时怎么优化，粉丝巨大的用户，推送耗时怎么优化？

### 进阶问题

- 推荐关注
- 用户搜索和内容搜索

## 系统实现

### 服务分解

* User Service: Register/Login
* Tweet Service: Post a tweet/News Feed/Timeline
* Media Service: Upload Picture/Video
* Friendship Service: Follow/Unfollow

### 数据存储

* User Service: SQL
* Tweet Service: NoSQL
* Media Service: File System
* Friendship Service: SQL/NoSQL

### 关键问题

#### News Feed 如何存取？

##### Pull模式

pull过程：
    
* 获取每个好友的前k条tweets，合并出k条news feed，K路归并算法：Merge K sorted arrays
* 假设有N个好友，则时间为 ==> N次DB Read的时间 + K路归并时间（可忽略）
* Post a tweet ==> 1次DB Write的时间

缺点：

* 读取慢（N次DB Reads，非常慢）
* 发生在用户获得News Feed的请求过程中，有延迟

##### Push模式

push过程：
    
* 每次News Feed，只用一次DB Read；
* 每次Post Tweet，会Fanout到N个Follower，需要N次DB Writes；
* 不过对于Post Tweet，可以用异步任务后台执行，用户无须等待

缺点：

* 异步写入可能碰到量太大的问题
* 删除微博需要同步数据

##### 推拉总结
    
目前主流的方式是用拉，把feed数据放到缓存里，根据时间来分片，采用多级缓存的方式解决拉过程多次db读写的问题。

#### 处理关注取消关注

* Follow之后：异步将他的Timeline合并到你的News Feed中
* Unfollow之后：异步将他的Tweets从你的News Feed中移除



参考资料

* [how-two-design-twitter-part-1](http://blog.gainlo.co/index.php/2016/02/17/system-design-interview-question-how-to-design-twitter-part-1/)

* [how-two-design-twitter-part-2](http://blog.gainlo.co/index.php/2016/02/24/system-design-interview-question-how-to-design-twitter-part-2/)

* [design-twitter-detail](https://segmentfault.com/a/1190000006165097)
