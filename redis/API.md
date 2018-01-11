## 2 redis API
### 2.1 全局键
	1. 查看所有键      keys *   注意: 会全局扫描  线上禁止使用  
	2. 查看键的总数    dbsize   直接获取 O(1)
	3. 检查键是否存在  exists key
	4. 删除键   del key
	5. 键过期   expire key  seconds  
	   ttl 可以查看 键还剩多少时间 
	6. 键的数据类型  type key 对外地常用5大类型  
	   内部类型可以使用 object encoding key  查看 
### 2.1.2 数据结构和内部编码

### 2.1.3 单线程架构
	1. 单线程架构 和 I/O 多路复用模型来实现高性能的内存数据库
    2. 所有客户端的命令都会进入到一个队列中，然后被执行。
		队列满了: 连接不上服务端？
		执行时间超时？
	3. 为什么快？  
		纯内存访问，
		非阻塞I/O, 使用epoll作为I/O多路复用技术的实现， 将Redis自身的事件模型将epoll中的连接、读写、关闭都转换为事件
		单线程避免了线程切换和竞态产生的消耗
	4.单线程的好处:  1.单线程可以简化数据结构和算法的实现 2.避免了线程切换和竞态产生的消耗
	问题是:对于每个命令的执行时间是有要求的。如果某个命令执行过长，会造成其他命令的阻塞
	
### 2.2 字符串 
#### 2.2.1 常用命令
	set key value [EX seconds] [PX milliseconds] [NX|XX]
	
	NX 键必须不存在才可以成功 ; XX 键必须存在才可以成功,用于更新
	NX 可以实现分布式锁，因为单线程命令处理机制，多个客户端执行set key value NX ,只有一个能成功。
	解锁必须删除键才可以，据说问题很多，分布式锁  可以使用 zookeeper 实现.
	
#### 2.2.2 内部编码
	·int：8个字节的长整型。
	·embstr：小于等于39个字节的字符串。
	·raw：大于39个字节的字符 
	
#### 2.2.3 场景
	1. 缓存 草稿
	2. 计数 生成编码
	3. 共享session 
	4. 限速  
	
### 2.3  哈希

### 2.3.1 命令
```
	127.0.0.1:6379> hset redisclient java jedis
	(integer) 1
	127.0.0.1:6379> hget redisclient
	(error) ERR wrong number of arguments for 'hget' command
	127.0.0.1:6379> hget redisclient java
	"jedis"
	127.0.0.1:6379> HSETNX  redisclient python pyredis
	(integer) 1
	127.0.0.1:6379> get redisclient
	(error) WRONGTYPE Operation against a key holding the wrong kind of value
	127.0.0.1:6379> hmget redisclent
	(error) ERR wrong number of arguments for 'hmget' command
	127.0.0.1:6379> hmget redisclent java python
	1) (nil)
	2) (nil)
	127.0.0.1:6379> hmget redisclient java python
	1) "jedis"
	2) "pyredis"
	127.0.0.1:6379> HEXISTS redisclient java
	(integer) 1
	127.0.0.1:6379> hkeys redisclient
	1) "java"
	2) "python"
	127.0.0.1:6379> HVALS redisclient
	1) "jedis"
	2) "pyredis"
	127.0.0.1:6379> hgetall redisclient
	1) "java"
	2) "jedis"
	3) "python"
	4) "pyredis"
	127.0.0.1:6379>
```

#### 2.3.2 内部编码
	ziplist(压缩列表)
	
	hashtable(哈希表)

#### 2.3.3 应用场景 
	缓存草稿 
	
### 2.4 列表
有序列表，可以存储 2^32-1 个元素。可以对列表 两端插入和 弹出,可以获取制定范围的元素列表。可以充当栈和队列

#### 2.4.1 命令

|操作类型|操 作|
|------|-----|
|添加  |rpush rpush linsert |
|查    |lrange  lindex  llen|
|删除  |lpop rpop lrem ltrim|
|修改  |lset                |
|阻塞操作|blpop brpop       |

#### 2.4.2 内部编码
·ziplist（压缩列表）
·linkedlist(链表)
 Redis3.2 提供了  quicklist 内部编码

#### 2.4.3 应用场景
1. 消息队列
2. 文章列表，数据导入
	 lpush+lpop=Stack（栈）
	·lpush+rpop=Queue（队列）
	·lpsh+ltrim=Capped Collection（有限集合）
	·lpush+brpop=Message Queue（消息队列）
	
### 2.5 集合 

和列表不同，不允许有重复的数据，可以用来求 交集，并集和差集

#### 2.5.1 使用场景 
典型使用场景为 标签，可以获得喜欢同一个标签的人,用户共同喜好

### 2.6 有序集合
和集合不同的是，可以排序，给每个元素设置一个分数,比较典型应用场景为排行榜系统。

### 2.7 键管理
 单个建，遍历键，数据库管理

#### 2.7.1 单个键 

type,del,object,exists,expire
1. 键重命名
rename  key  newkey 
2. 随机返回一个键
randomkey 
3. 键过期
可以设置键过期，查看剩余时间，可以删除过期，支持毫秒级。
expire key seconds：键在seconds秒后过期。
expireat key timestamp：键在秒级时间戳timestamp后过期。

redis 不支持二级结构(例如哈希、列表)内部元素的过期功能。
**4. 迁移键
migrate ,可以使用同步工具
*5. 遍历键
① keys *  会阻塞 
② scan 渐进式遍历，可以有效解决阻塞问题. 但是scan过程中有键的变化（增删改）,可能会遇到重复的键，scan并不能保证完整的遍历出来所有的键。


	
	
## 3.持久化
