# Feed流设计

Feed流本质上就是一个数据流，N个发布者的信息单元通过 关注关系 转发给M个接受者

需要看这个Feed流面相什么产品

关注关系：

- 单向 ：如果是单向那么存在大V效应，同时时效性可能低一些，比如分钟级别
- 双向：双向那么就是有好友，好友数量有限，例如微信 那么不存在大V效应

排序时间or推荐

- 目前大部分都是基于时间排序
- 推荐 类似于抖音根据用户的喜好匹配最佳的内容

支持用户量级：十万、百万、千万、上亿

| 类型   | 关注关系 | 是否有大V | 时序性 | 排序 |
| ------ | -------- | --------- | ------ | ---- |
| 微博   | 单向     | 有        | 秒/分  | 时间 |
| 抖音   | 单向/无  | 有        | 秒/分  | 推荐 |
| 朋友圈 | 双向     | 无        | 秒     | 时间 |
| 私信类 | 双向     | 无        | 秒     | 时间 |



# ==twitter Feed流设计与实现==

# 关注 relation  

微博最基本的就是关注和被关注。发动态、浏览动态、点赞、评论、收藏等



## 业务场景



attention 关注 && follower 粉丝

attention关注 可以关注和取关，follower粉丝 可以删除follower取消其他某个用户对自己的关注



## 业务特点

1. 海量用户 。亿级的用户数量 ，每个用户千级的帖子数量
2. 高访问量：每秒十万量级的平均页面访问，每秒万量级别的帖子发布
3. 用户分布的非均匀：部分用户的帖子数量/follower粉丝数量
4. 时间分布的非均匀：某个时间段用户的粉丝数量暴涨

一个典型社交系统：大数据量 高访问量 非均匀性



## DDL V1

因为业务场景需要实现 查看我关注了谁、取消关注、增加关注  

​                                                   有哪些粉丝、剔除黑粉

用户信息表 

user_info             id主键  user_id 用户信息 大V认证 头像 名称 注册时间 

user_realation    id主键 关注者id 被关注者id     A-》B 代表A关注B

查询用户信息

```bash
select * from user_info where user_id=xxx
```

查询关注了谁

```bash
select * from user_realation where attention_id=user_id_xxx
```

查询粉丝

```bash
select * from user_realation where follower_id=user_id_xxx
```

故只需要两张表即可查询出用户信息、粉丝数量、关注数量。但是随着用户增加,user_info表呈递增趋势增加以及user_relation表呈指数级别增加。

**优化一**

1. 进行水平拆分 将user_info表进行分片，比如将user_id按照range和hash取模一致性hash映射
2. 根据user_id分片查询用户信息仅可一次就可以，但是查询粉丝和关注那么就比较麻烦，例如可以根据user_id获取粉丝集合，假如relation表根据attention_id来进行分片，可以查询一次就可以获取关注数量。但是粉丝信息需要查询多个表。故当前环境下总有一个查询会比较耗时

**优化二：**

针对方案一 进行垂直拆分

## DDL V2

**优化二：**

针对方案一 进行垂直拆分

user_info 表 id user_id 用户相关信息 

attention 表 id  user_id attention_id

follower 表   id  user_id follower_id

将用户关系相关的表使用三张表来维护起来，这样根据user_id可以有利于分片，查询效率得到很大提升

查询关注

```bash
select attention_id from atention where user_id=xxx
```

查询粉丝

```bash
select follower_id from follower where user_id=xxx
```

查询用户

```bash
select * from user_info where id in(xxx)
```

即便上述简历索引，但是查询用户还是涉及多次IO处理

1. 对于某些大V用户，查看粉丝数量并进行查询用户信息，需要在user_id扫描行数仍然很多
2. 粉丝数量较多是，进行分页查询效率较慢

## 引入缓存

粉丝数量和关注数量 count(*)优化：同时在DB层，user_info信息表当中添加关注数量和粉丝数量，因为count(1000万数据时间很长)

优化一：将热点户信息引入缓存当中，在feed页面realation摘要当中，根据用户作为key来查询缓存，如果缓存查不到在进行查询DB

优化二：引入本地缓存，在服务器端进行本地缓存，redis作为二级缓存，因为redis作为缓存可能涉及大量网络IO情况。二级缓存可能仅针对一些大V用户，本地缓存设置一个不长的过期时间

优化三：缓存一致性问题 增加一个用户先写redis->异步去写DB



## 关注

一个用户的关注数量不会太多，差不多在1000以内左右

所以直接将用户的关注id 直接放在redis list链表当中

## 粉丝

关注场景因为用户数量少，但是粉丝数量存在大V等情况，

方案一：如果全放在redis list当中那么是不合适的，查询效率将会很慢，所以用redis list作为缓存不使用。

方案二：增量化的实现解决方案：新增的粉丝用户是放到最前边的，并且往往前N页查询比较频繁。所以新增的粉丝放在redis list当中，并且可以适当存放前五页的粉丝放在list当中。后续的粉丝可以查询DB



# 发布帖子

## 初版DDL

一般存储帖子在mysql中进行存储，一般有如下几个字段：

帖子表：post_id,user_id,post_time,contnet

- post_id:代表发布帖子的ID
- user_id:代表谁发的帖子
- post_time:发布时间
- content:发布内容

## 场景

场景一：查询特定帖子内容

```bash
select * from post_blog where post_id=xxx
```

场景二：查询某个用户的发布的贴子内容

```bash
select * from post_blog where user_id=xxx and post_time>=?
```

## DDL优化

对于上述的查询 有一个严重问题那么就是回表（对于二级索引查询到的每条记录都需要到聚族索引当中进行查询）从而降低DB吞吐量，所以讲user_id和post_time冗余到post_id当中

- post_id=user_id(6位) + post_time(6位)+seq(2位) 

- user_id:0-9/A-Z 36进行来代表用户id

- post_time: 后6位来表示秒 可以表示70年范围内任意一秒   

- seq:  单个用户每秒发放的帖子数量不超过2个bit位

这样设计之后就可以根据发布帖子的时间自增长 从而连续递增，可以充分利用到DB的范围扫描

查询某个用户发送的帖子场景

```bash
select * from post where postID between postID1 and postID2 
```

或者

```bash
select * from post where postID LIKE 'XXX%'
```

优化：

1. 但是随着帖子数量增加，单机模式下进行水平拆分，根据user_id进行拆分
2. 读写分离
   1. 数据延迟
   2. 存储效率低下，7天以前的帖子可能很少被访问

## 引入缓存

由于用户帖子超过七天之前的可能大量不会被访问

redis key=user_id+时间戳（精确到星期）  value hash类型 key=post_id value=content  expire设计过期时间

所以对某个用户一段时间查找变为用户针对本周时间戳的hscan命令 ，**用户发帖的同时同步更新DB和缓存**，DB变更操作记录保持一致性



## 热点用户帖子

某个大V粉丝数量可能很多，意味着这个redis所在服务器可能查询在1000万QPS/S，这样可以进行进一步增加本地缓存来实现





# 点赞业务设计

根据点赞业务的贴点可以发现

1. 高吞吐量
2. 可以接受数据不一致

可以用Mysql来做固定存储，redis来做缓存，读写操作都落缓存，异步线程定期刷新DB。大部分其他业务也是这么做的 秒杀、商品库存

## DDL版本一

mysql存储： post_id 点赞、转发、评论、阅读量

redis存储:

post_id+点赞 value个数

post_id+转发 value个数

post_id+评论 value个数

....

微博的业务难点是在业务上的扩展性问题，以及效率问题。



## DDL版本二优化

mysql存储： post_id key value    mysql就是以键值对方式来进行存储

​					    post_id key value 



redis缓存： key=post_id  value: 点赞_转发 评论 阅读量 按照一定数据格式进行存储redis当中





## redis源码 点赞业务优化

redis如果以字符串进行存储的话，即便redis里边存在sds不同长度的数据结构 但还是比较浪费空间的呢。

而且redis 字符串操作 如果value是整数 就是尽量使用整数来做

修改redis源码

```c
struct item{
  int64_t id; 
  int start_num; //点赞数 
  int repost_num; //转发数
  int comment_num //评论数
}
```

而针对业务场景 还可以对上述进行优化 因为转发数和评论数相比较于低  类似于csdn 掘进 

```c
struct item{
  int64_t id; 
  int start_num 
  short repost_num
  short comment_num 
}
```



# 大V发帖子，关注如何实现落地 重点

例如微博、掘进、抖音关注用户之后，当第一时间点看关注就可以获取到关注者的动态 一般以时间进行排序。关注的人更新了帖子 我们就是会看到。

随着用户的relation变化，某个用户的timeline也会随之变化。而一个用户帖子的增删也会影响其他用户的timeline。当relation关系复杂时，timeline的性能也会收到挑战。

举个例子：

现关注：腾讯新闻、爱奇艺、新浪、头条  而刷新出现  腾讯内容新闻、爱奇艺内容、新浪内容、头条内容

新增关注：  抖音                                              刷新时出现 抖音、腾讯、爱奇艺、抖音、新浪、头条    这里边抖音发布两条

删除关注： 爱奇艺                                           刷新时出现 抖音、腾讯、抖音、新浪、头条 

## 实现思路

经典实现思路两种模式 push模式和pull模式



## DDL

timeline表  id  user_id  post_id  post_time  

user_id:重push模式 user_id就是代表follower_id 

post_id：代表文章id user_id和post_id唯一约束

post_time: 发布时间

## push模式 写扩散

push模式特点:用户每发一个帖子 ，将此帖子推送到follower所在分片上，follower在每次浏览timeline时，直接查询自己的存储内容

查询timeline表 即可获取文章

```bash
select  post_id from timeline where user_id=xxx and post_time between A and B
select * from post where post_id in(...) 
```

用户关注了新用户 将新用户post_id获取 并插入到timeline表中

用户取关了              delete from timeline where user_id=本用户 and post_id like 'xxx%' xxx代表取关用户ID



push模式 写扩散缺点

1. 假设B是热点用户，粉丝数量上千万，那么每新增一次帖子或者删除一次帖子都将会上千万的复制。
2. 大V用户下，可能存在大量僵尸粉，针对僵尸粉发送信息可能造成存储浪费

**但是微信 wechat 可以利用push写扩散这种模式 因为好友用户的数量是有限的呢**

以微信举例子：如何记录用户上一次刷新朋友圈时间

以抖音举例子：如何记录用户上一次退出抖音软件时间，以及长时间未登录 这里边是否存在优化呢



个人观点 push模式更像是回调机制，就像etcd分布式锁

## pull模式 读扩散

与push模式不同，pull模式每次新增/删除帖子不用同步所有followers ，

insert timeline 可以查询关注列表 假如关注了用户A和用户B

在根据用户A和用户B的user_id 去 post帖子表中去查询最新的帖子列表 post_time

优点：

1. pull的模式简单，并不需要维护timeline表 

缺点：

1. 假如有100个DB分片，那么每个用户查询关注需要扫描100次DB （100万人 每10秒刷新一次）
2. 最大问题是每个粉丝需要记录到上次读到关注者那条信息，如果有1000个关注者，那么当前用户需要记录1000个位置信息，这个量是和关注量成正比的，远比用户量要大的多。
3. 

针对缺点进行优化 在电商场景下，假如redis缓存失效了，大量请求都打入到db当中，把这些读操作都整合到一个队列进行合并并发结果进行返回。



个人观点，pull模式更像是主动轮训 ，就像redis分布式锁

## push和pull集合

push：大部分用户消息是写扩散 也即 就是push模式

pull:    针对大V这种是读扩散 

这样既解决写扩散资源浪费，读扩散QPS峰值过高



## 加入缓存

这里边应该存在两种模式：主动推送 被动刷新

- 方案一：主动推送模式

当关注者用户发布消息，关注页面 关注字段可以展示红点来表示主动推送模式。

服务端维护一个session池子，记录那些用户在线，当用户A发送了一个帖子，找到用户A的follower 去判断session池子中有哪些粉丝正好在线，告知粉丝有新消息了，在关注界面上显示红点来表示有新消息了。

- 方案二：被动刷新

客户端周期性刷新 

可以根据timeline进行增量列表的添加。可以以每10秒内发帖用户的user_id维护在一个list当中

297439122  用户id  1,2,3,4,5

297439132  用户id  6,7,8,9,10

297439142  用户id  11,12,13,14        同一个列表中的ID 以B+树形式来展示

作为缓存，增量查询不会保留数据，但是默认为每个用户保留上一次的查询结果，可以认为增量数据不会查询太老的结果。



# Feed流架构和设计落地







# ==朋友圈==

朋友圈也是典型的feed流，并且用户关系是双向的 是一种典型的写扩散





