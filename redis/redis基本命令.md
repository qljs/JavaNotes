



## 一. Linux安装Redis

```sh
1. 下载Redis上传至服务器，
或者用命令下载wget http://download.redis.io/releases/***.tar.gz

2. 解压redis
tar -zxvf redis-*.tar.gz

3. 安装C语言包
yum install -y gcc

4. 进入redis-*目录下进行安装（makefile是安装文件）
make 
# 指定安装目录
make install PREFIX=/usr/redis

5. 修改配置文件
mv /redis源码目录中redis.conf /redis安装目录
#修改 后台启动
daemonize yes  
 #关闭保护模式，开启的话，只有本机才可以访问redis
protected-mode no 
# 需要注释掉bind
# bind 127.0.0.1（bind绑定的是自己机器网卡的ip，如果有多块网卡可以配多个ip，代表允许客户端通过机器的哪些网卡ip去访问，内网一般可以不配置bind，注释掉即可）

6. 启动服务
src/redis-server redis.conf
```



**Info：查看redis服务运行信息，分为 9 大块，每个块都有非常多的参数，这 9 个块分别是:** 

Server 服务器运行的环境参数 

Clients 客户端相关信息 

Memory 服务器运行内存统计数据 

Persistence 持久化信息 

Stats 通用统计数据 

Replication 主从复制相关信息 

CPU CPU 使用情况 

Cluster 集群信息 

KeySpace 键值对统计数量信息



## 二. Redis常用的数据结构

### 1. String

**String数据结构是简单的key-value类型，value不仅可以是String，也可以是数字，因此可以用来做计数器，统计一些信息。**

```sh
1. SET key value
#-作用：添加一个k-v。
#-返回值：成功返回OK。

2. GET  key
#-作用：查找key
#-返回值：成功返回value，不存在返回nil。

3. SETNX key value
#-作用：KEY不存在时，将key的值设value,当key存在是，不做任何动作。
#-返回值：成功返回1，失败返回0。

4. SETEX命令/PSETEX命令（原子（atomic）操作）
#-作用：KEY不存在时，将key的值设value并将设置生存时间,当key存在是，不做任何动作。
#-返回值：成功返回1，失败返回0。

5. GETSET key value
#-作用：把key值设为value，并返回旧值。
#-返回值：成功返回旧值，不存在返回nil，key存在但不是字符串时，返错误。

6. APPEN key value
#-作用：如果key存在，在旧value后面追加新value，若不存在直接设为新value。
#-返回值：返回VALUE长度。

7. SETRANGE key offset value
#-作用：从偏移量offset开始， 用value参数覆写(overwrite)键key储存的字符串值。
#-返回值：返回被修改之后， 字符串值的长度。

8. GETRANGE key start end
#作用：返回键 key 储存的字符串值的指定部分， 字符串的截取范围由start和end两个偏移量决定 (包括start和end 在内)。
#返回值：返回字符串值的指定部分。

9. INCR key/INCRBY key increment
#-作用：value加1/increment，若key不存在，value初始化为0，再加上1/increment，如果键key储存的值不能被解释为数字， 那么INCR命令将返回一个错误。
#-返回值：返回字value的值。

11. DECR key/DECRBY key decrement
#-作用：value减1/decrement，若key不存在，初始化为0再减1/decrement，如果键key储存的值不能被解释为数字， 那么INCR命令将返回一个错误。
#-返回值：返回字value的值。

12. MSET key value [key value …]
#-作用：给多个key设值
#-返回值：返回字value的值。
```



### 2. hash

**hash是一个String类型的field和value的映射表，hash适合存储对象。**

```shell
1. HSET hash field value
#-作用：将hash中field的值设，置为values。
#-返回值：field不存在时返回1，存在是返回0。

2. HGET hash field
#-作用：返回哈希表中给定域的值。
#-返回值：field不存在时返回1，存在是返回0。

3. HSETNX hash field value
#-作用：feild不存在时，值设置为value，若field存在，放弃操作员。
#-返回值：field不存在时返回1，存在是返回0。

4. HEXISTS hash field
#-作用：检查hash中是否存在field。
#-返回值：field不存在时返回1，存在是返回0。

5. HDEL key field [field …]
#-作用：删除哈希表key中的一个或多个指定域，不存在的域将被忽略。
#-返回值：被成功移除的域的数量，不包括被忽略的域。

6. HLEN key
#-作用：返回哈希表key中域的数量。
#-返回值：哈希表中域的数量，key不存在返回0。

7. HSTRLEN key field
#-作用：返回哈希表key中feild的长度。
#-返回值：feild的长度，field不存在返回0。

8. HINCRBY/HINCRBYFLOAT key field increment
#-作用：field的值加上整数/浮点数increment，若key不存在，创建并加increment，若field不存在，则创建并初始化为0，若filed值为字符串，则会报错。
#-返回值：feild的新值。

9. HMSET key field value [field value …]
#-作用：给key设置多个值，此命令会覆盖hash表中已存在的field。
#-返回值：执行成功返回OK。

10. HMGET key field [field …]
#-作用：查找hash表中一个或多个field的值。
#-返回值：一个包含多个给定域的关联值的表，表值的排列顺序和给定域参数的请求顺序一样。

11. HKEYS key
#-作用：返回哈希表key中所有的field。
#-返回值：一个包含哈希表中所有域的表。

12. HVALS key
#-作用：返回哈希表key中所有的field。
#-返回值：一个包含哈希表中所有域的表。

12. HGETALL key
#-作用：返回哈希表key中所有的field和filed的值。
#-返回值：以列表形式返回哈希表的域和域的值。
```



### 3. list

**list是是一个双向链表，支持反向查找和遍历，可以用来关注列表、队列（LPUSH + RPOP）、阻塞队列等。**

```sh
1. LPUSH key value [value …]
#-作用：将一个或多个值插入到列表key的表头，多个值时从左到右，列表不存在时，新建列表。
#-返回值：列表的长度。

2. LPUSHX key value
#-作用：将value插入列表表头，当列表不存在时不作任何操作。
#-返回值：列表新的长度。

3. RPUSH key value [value …]
#-作用：讲一个或多个值插入表尾，顺序从左到右，列表不存在时，会新建列表。
#-返回值：列表新的长度。

4. RPUSHX key value
#-作用：讲一个个值插入表尾，当列表不存在时不作任何操作。
#-返回值：列表新的长度。

5. LPOP key
#-作用：移除并返回列表的头元素。
#-返回值：列表的头元素。

5. RPOP key
#-作用：移除并返回列表的尾元素。
#-返回值：列表的尾元素。

6. RPOPLPUSH source destination
#-作用：在一个原子时间内，先将列表source中最后一个元素弹出，然后将弹出元素插入列表destination的头，当source与destination相同时，会将元素放入头部。
#-返回值：被弹出的元素。

7. LREM key count value
#-作用：根据参数count的值，移除列表中与参数value相等的元素。
- count 的值可以是以下几种：
- count > 0 : 从表头开始向表尾搜索，移除与 value 相等的元素，数量为 count 。
- count < 0 : 从表尾开始向表头搜索，移除与 value 相等的元素，数量为 count 的绝对值。
count = 0 : 移除表中所有与 value 相等的值。
#-返回值：被移除元素的数量。

8. LLEN key
#-作用：返回列表的长度。
#-返回值：列表的长度 。

9. LINDEX key index
#-作用：返回列表中下标为index的的元素。
#-返回值：列表中下标为index的的元素。

10 LINSERT key BEFORE|AFTER pivot value
#-作用：将value插入列表pivot前|后。
#-返回值：列表的新长度。

11 LSET key index value
#-作用：将列表index位置的值设置为value。
#-返回值：成功返回ok。

12. LRANGE key start stop
#-作用：查找列表中指定区间内(闭区间)的元素,index从0开始,-1表示列表的最后一个元素。
#-返回值：成功返回ok。

13. LTRIM key start stop
#-作用：删除列表中指定区间外(闭区间)的元素,。
#-返回值：成功返回ok。

14. BLPOP key [key …] timeout
#-作用：阻塞弹出头元素，直到等到超时或有可弹出元素。
#-返回值：返回弹出元素。

15. BRPOP key [key …] timeout
#-作用：阻塞弹出最后一个元素，直到等到超时或有可弹出元素。
#-返回值：返回弹出元素。

16. BRPOPLPUSH source destination timeout
#-作用：阻塞弹出最后一个元素至另一个列表，直到等到超时或有可弹出元素。
#-返回值：返回弹出元素。
```



### 4.  set

**set的功能与list相似，但是set可以自动排序以及去重，可以应用在点赞 数、收藏、共同关注等。**

```shell
1. SADD key member [member …]
#-作用：将一个或多个元素加入集合中，集合已经存在的将被忽略。
#-返回值：添加到集合中的新元素的数量。

2. SISMEMBER key member
#-作用：判断集合中是否包含元素。
#-返回值：包含返回1，怒存在返回0。

3. SPOP key [count]
#-作用：移除并返回集合中的一个(不带count)/或多个元素随机元素。
#-返回值：被移除的随机元素，不存在时返回nil。

4. SRANDMEMBER key [count]
#-作用：返回集合中的一个/或多个元素随机元素，当count为正数时切大于集合元素数时，返回整个集合，当count为负数时，返回count绝对值的元素个数，但是可能会有重复元素。
#-返回值：随机元素，不存在时返回nil。

5. SREM key member [member …]
#-作用：移除制定的一个或多个元素，key不是集合时返回错误。
#-返回值：被移除的元素数量。

6. SMOVE source destination member（原子操作）
#-作用：把member从source移到destination。
#-返回值：成功移除返回1，若member不存在返回0。

7. SCARD key
#-作用：返回集合的元素数量。
#-返回值：集合中的元素数量。

8. SMEMBERS key
#-作用：返回集合中所有元素。
#-返回值：集合中所有元素。

9. SINTER key [key …]
#-作用：查找一个集合的全部成员，或多个集合的相同元素，交集。
#-返回值：交集成员的列表。

10. SINTERSTORE destination key [key …]
#-作用：将一个集合的全部成员，或多个集合的相同元素复制到destination集合。
#-返回值：结果集中的成员数量。

11. SUNION key [key …]
#-作用：查找一个集合全部元素，或多个集合的并集。
#-返回值：结果集中的成员数量。

12. UNIONSTORE destination key [key …]
#-作用：将一个集合的全部成员，或多个集合的并集复制到destination集合。
#-返回值：结果集中的成员数量。

13. SDIFF key [key …]
#-作用：查找一个集合全部元素，或多个集合的差集。
#-返回值：包含差集成员的列表。

14. SDIFFSTORE destination key [key …]
#-作用：将一个集合的全部成员，或多个集合的差集复制到destination集合。
#-返回值：结果集中的元素数量。
```



### 5. zset

**与set相比，sorted set增加了一个权重参数score,使集合中的元素可以按照score进行排序，例如微博的热搜等。**

```shell
1. ZADD key score member [[score member] [score member] …]
#-作用：向集合中添加一个或多个元素。
#-返回值：新成员的数量。

2. ZSCORE key member
#-作用：查找集合中元素的score。
#-返回值：score的值。

3. ZINCRBY key increment member
#-作用：给集合中元素score加上increment。
#-返回值：新score的值。

4. ZCARD key
#-作用：查找集合中元素数量。
#-返回值：集合中元素数量。

5. ZCOUNT key min max
#-作用：查找集合中score在[min,max]区间内元素的数量。
#-返回值：元素的数量。

6. ZRANGE|ZREVRANGE key start stop [WITHSCORES]
#-作用：查找集合中index在[min,max]区间内的元素，WITHSCORES:显示scores，并按scores从小到大排序(ZRANGE)|并按scores从大到小排序(ZREVRANGE)。
#-返回值：区间内的元素的。

8. ZRANGEBYSCORE| key min max [WITHSCORES] [LIMIT offset count]
#-作用：查找集合中scores在[min,max]区间内的元素，WITHSCORES:显示scores，并按scores从小到大排序(ZRANGEBYSCORE)|并按scores从大到小排序(ZREVRANGEBYSCORE )， limit:index从offset开始，共cout个元素，可以使用()将闭区间改为开区间。
#-返回值：区间内的元素的。

10. ZRANK|ZREVRANK  key member
#-作用：查找集合中元素的下标，从小到大排名(ZRANK)|从大到小排名(ZREVRANK)。
#-返回值：集合中元素的下标。

11. ZREM key member [member …]
#-作用：删除集合中指定的一个或多个元素。
#-返回值：成功移除的成员的数量。

12. ZREMRANGEBYRANK key start stop
#-作用：删除集合中下表在[start, stop]的元素。
#-返回值：被移除成员的数量。

13. ZREMRANGEBYSCORE key min max
#-作用：删除集合中scores在[min, max]的元素。
#-返回值：被移除成员的数量。

14. ZRANGEBYLEX key '['|'('min '['|'('max [LIMIT offset count]
#-作用：查找scores在[|(min至 [|(max区间内，下标从offset开始，共count个元素,'+'表示比指定元素大的，‘-’表示比指定元素小的。
#-返回值：区间内的元素。
#-例子:
>zrangebylex myzset [mem0 [mem4 limit 0 5
1) "mem0"
2) "mem1"
3) "mem2"
4) "mem3"
5) "mem4"
>zrangebylex myzset [mem0 +  limit 0 5
1) "mem0"
2) "mem1"
3) "mem2"
4) "mem3"
5) "mem4"

15. ZLEXCOUNT key min max
#-作用：查找scores在[|(min至 [|(max区间内，下标从offset开始，共count个元素,'+'表示比指定元素大的，‘-’表示比指定元素小的。
#-返回值：元素的数量。

16. ZREMRANGEBYLEX key min max
#-作用：移除scores在[|(min至 [|(max区间内的元素t。
#-返回值：被移除元素的数量。

17. ZUNIONSTORE|ZINTERSTORE  destination numkeys key [key …] [WEIGHTS weight [weight …]] [AGGREGATE SUM|MIN|MAX]
#-作用：计算给定的一个或多个有序集的并集|交集，集合的个数由numkeys指定，并将结果放入destination，
#-	   WEIGHTS：每个集合中scores乘以对应weight，默认乘以1。
#-     AGGREGATE：默认参数SUM（集合中相同元素的scores相加），MIN(相同元素取最小的scores)，MAX(相同元素取最大的scores)
#-返回值：被移除元素的数量。
```



## 三. Redis 配置文件主要配置

```sh
################################## 网络 #####################################
bind 127.0.0.1   # 绑定的IP
port 6379 # 端口
protected-mode yes # 保护模式，默认开启，只能本机访问

################################# 通用 #####################################
pidfile /var/run/redis_6379.pid # redis后台运行时指定
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel notice # 日志级别
logfile "" # 日志文件位置名称
databases 16 # redis数据库数量
always-show-logo yes # 启动时是否显示logo

################################ 快照  ################################
# save <seconds> <changes> seconds时间内至少一个key发生changes次改变进行持久化
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes # 持久化报错后主进程fork是否继续工作，yes：是
rdbcompression yes # 是否压缩rdb文件
rdbchecksum yes # 保存rdb文件是否校验
dir ./ # rdb文件目录
dbfilename dump.rdb #rdb文件名称

################################# 主从复制 #################################
# 设置主节点 ip   host
replicaof <masterip> <masterport>
masterauth <master-password> # 主节点密码
replica-read-only yes  # 子节点是否只读

################################### 客户端 ####################################
maxclients 10000 #最大连接数

################################## 安全 ###################################
# 查看秘密：config get requirepass 
# 设置密码：config set requirepass ""
# 登陆：auth [密码]
requirepass foobared # 密码

############################## 内存管理 ################################
maxmemory <bytes> # 最大内存
# volatile-lru -> 从设置了过期时间的key中移除最近最少使用的；
# allkeys-lru -> 内存不足时移除最近最少使用的key.
# volatile-lfu -> 从设置过期时间的key中移除最少使用的.
# allkeys-lfu -> 内存不足时移除最少使用的key.
# volatile-random -> 从设置过期时间的key中随即删除.
# allkeys-random -> 内存不足时随机删除.
# volatile-ttl -> 从设置过期时间的key中删除即将过期的.
# noeviction -> 内存不足时报错.
maxmemory-policy noeviction # 内存满了之后处理策略

############################## AOF持久化 ###############################
appendonly no # 是否开启aof模式，默认不开启
appendfilename "appendonly.aof" # aof文件名字
#appendfsync always # 每次发生变化都写入aof文件，会影响redis性能
appendfsync everysec # 每秒钟写入一次，可能会丢失一秒内的数据
#appendfsync no # 由系统决定写入
```

