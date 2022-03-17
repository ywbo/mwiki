#  Redis常用命令

## 一、redis启动：

>     本地启动：redis-cli
>     远程启动：redis-cli -h host -p port -a password

### 1. Redis 连接命令

```sql
AUTH password  -- 验证密码是否正确
```

```sql
ECHO message   -- 打印字符串
```

```sql
PING -- 查看服务是否运行
```

```sql
QUIT  -- 关闭当前连接
```

```sql
SELECT index -- 切换到指定的数据库（index指的是序号0,1,2,3,4...）
```

## 二、Redis 整体查看

### 1. 查询数据库数量

```sql
127.0.0.1:6379> CONFIG GET databases
1) "databases"
2) "16"
```

### 2. 查看key的数量

```sql
127.0.0.1:6379> INFO keyspace
# Keyspace
db0:keys=28,expires=0,avg_ttl=0
db1:keys=1,expires=0,avg_ttl=0
db2:keys=1,expires=0,avg_ttl=0
```

## 三、Redis `keys` 命令

```sql
DEL key  -- 删除数据库
```

```sql
DUMP key  -- 序列化给定的key并返回序列化的值
```

```sql
EXISTS key  --检查给定的key是否存在
```

```sql
EXPIRE key seconds  -- 为key设置过期时间
```

```sql
EXPIRE key timestamp  -- 用时间戳的方式给key设置过期时间
```

```sql
PEXPIRE key milliseconds  -- 设置key的过期时间以毫秒计
```

```sql
KEYS pattern  -- 查找所有符合给定模式的key
```

```sql
MOVE key db  -- 将当前数据库的key移动到数据库db当中
```

```sql
PERSIST key  -- 移除key的过期时间，key将持久保存
```

```sql
PTTL key  -- 以毫秒为单位返回key的剩余过期时间
```

```sql
TTL key  -- 以秒为单位，返回给定key的剩余生存时间
```

```sql
RANDOMKEY  -- 从当前数据库中随机返回一个key
```

```sql
RENAME key newkey  -- 修改key的名称
```

```sql
RENAMENX key newkey  -- 仅当newkey不存在时，将key改名为newkey
```

```sql
TYPE key  -- 返回key所存储的值的类型
```



## 四、Reids `字符串` 命令

```sql
1、SET key value  -- 添加数据库
```

```sql
2、GET key  -- 获取
```

```sql
GETRANGE key start end  -- 返回key中字符串值的子字符
```

```sql
GETSET key value  -- 将给定key的值设为value，并返回key的旧值
```

```sql
GETBIT KEY OFFSET  -- 对key所储存的字符串值，获取指定偏移量上的位
```

```sql
MGET KEY1 KEY2 -- 获取一个或者多个给定key的值
```

```sql
SETBIT KEY OFFSET VALUE  -- 对key所是存储的字符串值，设置或清除指定偏移量上的位
```

```sql
SETEX key seconds value  -- 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。
```

```sql
SETNX key value  -- 只有在 key 不存在时设置 key 的值。
```

```sql
SETRANGE key offset value  -- 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。
```

```sql
STRLEN key  -- 返回 key 所储存的字符串值的长度。
```

```sql
MSET key value [key value ...]  -- 同时设置一个或多个 key-value 对。
```

```sql
MSETNX key value [key value ...]   -- 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。
```

```sql
PSETEX key milliseconds value  -- 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。
```

```sql
INCR key   -- 将 key 中储存的数字值增一。
```

```sql
INCRBY key increment  -- 将 key 所储存的值加上给定的增量值（increment） 。
```

```sql
INCRBYFLOAT key increment  -- 将 key 所储存的值加上给定的浮点增量值（increment） 。
```

```sql
DECR key -- 将 key 中储存的数字值减一。
```

```sql
DECRBY key decrement  -- key 所储存的值减去给定的减量值（decrement） 。
```

```sql
APPEND key value  -- 如果 key 已经存在并且是一个字符串， APPEND 命令将 指定value 追加到改 key 原来的值（value）的末尾。
```

## 五、Redis `hash` 命令

```sql
HDEL key field1 [field2]  -- 删除一个或多个哈希表字段
```

```sql
HEXISTS key field  -- 查看哈希表 key 中，指定的字段是否存在。
```

```sql
HGET key field  -- 获取存储在哈希表中指定字段的值。
```

```sql
HGETALL key   -- 获取在哈希表中指定 key 的所有字段和值
```

```sql
HINCRBY key field increment  -- 为哈希表 key 中的指定字段的整数值加上增量 increment 。
```

```sql
HINCRBYFLOAT key field increment  -- 为哈希表 key 中的指定字段的浮点数值加上增量 increment 。
```

```sql
HKEYS key  -- 获取所有哈希表中的字段
```

```sql
HLEN key   -- 获取哈希表中字段的数量
```

```sql
HMGET key field1 [field2]  -- 获取所有给定字段的值
```

```sql
HMSET key field1 value1 [field2 value2 ]   -- 同时将多个 field-value (域-值)对设置到哈希表 key 中。
```

```sql
HSET key field value   -- 将哈希表 key 中的字段 field 的值设为 value 。
```

```sql
HSETNX key field value   -- 只有在字段 field 不存在时，设置哈希表字段的值。
```

```sql
HVALS key   -- 获取哈希表中所有值
```

```sql
HSCAN key cursor [MATCH pattern][COUNT count]   -- 迭代哈希表中的键值对。
```

## 六、Redis `列表`命令

```sql
BLPOP key1 [key2 ] timeout   -- 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
```

```sql
BRPOP key1 [key2 ] timeout   -- 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
```

```sql
BRPOPLPUSH source destination timeout    ---  从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
```

```sql
LINDEX key index   -- 通过索引获取列表中的元素
```

```sql
LINSERT key BEFORE|AFTER pivot value   -- 在列表的元素前或者后插入元素
```

```sql
LLEN key   -- 获取列表长度
```

```sql
LPOP key  -- 移出并获取列表的第一个元素
```

```sql
LPUSH key value1 [value2]   -- 将一个或多个值插入到列表头部
```

```sql
LPUSHX key value  -- 将一个值插入到已存在的列表头部
```

```sql
LRANGE key start stop   -- 获取列表指定范围内的元素
```

```sql
LREM key count value -- 移除列表元素
```

```sql
LSET key index value  -- 通过索引设置列表元素的值
```

```sql
LTRIM key start stop  -- 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
```

```sql
RPOP key   -- 移除并获取列表最后一个元素
```

```sql
RPOPLPUSH source destination  -- 移除列表的最后一个元素，并将该元素添加到另一个列表并返回
```

```sql
RPUSH key value1 [value2]  -- 在列表中添加一个或多个值
```

```sql
RPUSHX key value   -- 为已存在的列表添加值
```

## 七、Redis `集合`命令

```sql
SADD key member1 [member2]  -- 向集合添加一个或多个成员
```

```sql
SCARD key   -- 获取集合的成员数
```

```sql
SDIFF key1 [key2]   -- 返回给定所有集合的差集
```

```sql
SDIFFSTORE destination key1 [key2]   -- 返回给定所有集合的差集并存储在 destination 中
```

```sql
SINTER key1 [key2]   -- 返回给定所有集合的交集
```

```sql
SINTERSTORE destination key1 [key2]   -- 返回给定所有集合的交集并存储在 destination 中
```

```sql
SISMEMBER key member  -- 判断 member 元素是否是集合 key 的成员
```

```sql
SMEMBERS key --- 返回集合中的所有成员
```

```sql
SMOVE source destination member  -- 将 member 元素从 source 集合移动到 destination 集合
```

```sql
SPOP key  -- 移除并返回集合中的一个随机元素
```

```sql
SRANDMEMBER key [count]  -- 返回集合中一个或多个随机数
```

```sql
SREM key member1 [member2]  -- 移除集合中一个或多个成员
```

```sql
SUNION key1 [key2]  -- 返回所有给定集合的并集
```

```sql
SUNIONSTORE destination key1 [key2]   -- 所有给定集合的并集存储在 destination 集合中
```

```sql
SSCAN key cursor [MATCH pattern][COUNT count]   -- 迭代集合中的元素
```

## 八、Redis `有序集合`命令

```sql
ZADD key score1 member1 [score2 member2]   -- 向有序集合添加一个或多个成员，或者更新已存在成员的分数
```

```sql
ZCARD key   -- 获取有序集合的成员数
```

```sql
ZCOUNT key min max   -- 计算在有序集合中指定区间分数的成员数
```

```sql
ZINCRBY key increment member -- 1有序集合中对指定成员的分数加上增量 increment
```

```sql
ZINTERSTORE destination numkeys key [key ...] -- 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中
```

```sql
ZLEXCOUNT key min max  -- 在有序集合中计算指定字典区间内成员数量
```

```sql
ZRANGE key start stop [WITHSCORES]  -- 通过索引区间返回有序集合成指定区间内的成员
```

```sql
ZRANGEBYLEX key min max [LIMIT offset count]   -- 通过字典区间返回有序集合的成员
```

```sql
ZRANGEBYSCORE key min max [WITHSCORES][LIMIT]   -- 通过分数返回有序集合指定区间内的成员
```

```sql
ZRANK key member   -- 返回有序集合中指定成员的索引
```

```sql
ZREM key member [member ...]   -- 移除有序集合中的一个或多个成员
```

```sql
ZREMRANGEBYLEX key min max   -- 移除有序集合中给定的字典区间的所有成员
```

```sql
ZREMRANGEBYRANK key start stop   -- 移除有序集合中给定的排名区间的所有成员
```

```sql
ZREMRANGEBYSCORE key min max  -- 移除有序集合中给定的分数区间的所有成员
```

```sql
ZREVRANGE key start stop [WITHSCORES]  -- 返回有序集中指定区间内的成员，通过索引，分数从高到底
```

```sql
ZREVRANGEBYSCORE key max min [WITHSCORES]   -- 返回有序集中指定分数区间内的成员，分数从高到低排序
```

```sql
ZREVRANK key member  -- 返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序
```

```sql
ZSCORE key member  -- 返回有序集中，成员的分数值
```

ZUNIONSTORE destination numkeys key [key ...]  -- 计算给定的一个或多个有序集的并集，并存储在新的 key 中

```sql
ZSCAN key cursor [MATCH pattern][COUNT count] -- 迭代有序集合中的元素（包括元素成员和元素分值）
```

## 九、Redis `发布订阅`命令

```sql
PSUBSCRIBE pattern [pattern ...]  -- 订阅一个或多个符合给定模式的频道。
```

```sql
PUBSUB subcommand [argument [argument ...]]  -- 查看订阅与发布系统状态。
```

```sql
PUBLISH channel message  -- 将信息发送到指定的频道。
```

```sql
PUNSUBSCRIBE [pattern [pattern ...]]  -- 退订所有给定模式的频道。
```

```sql
SUBSCRIBE channel [channel ...]  -- 订阅给定的一个或多个频道的信息。
```

```sql
UNSUBSCRIBE [channel [channel ...]]  -- 指退订给定的频道。
```

示例：

```sql
redis 127.0.0.1:6379> SUBSCRIBE redisChat

Reading messages... (press Ctrl-C to quit)
 "subscribe"
 "redisChat"
 (integer) 1
```

现在，我们先重新开启个 redis 客户端，然后在同一个频道 redisChat 发布两次消息，订阅者就能接收到消息。

```sql
redis 127.0.0.1:6379> PUBLISH redisChat "Redis is a great caching technique"
(integer) 1
```

订阅者的客户端会显示如下消息

```sql
 "message"
 "redisChat"
 "Redis is a great caching technique"
```

## 十、Redis `事务`命令

```sql
DISCARD -- 取消事务，放弃执行事务块内的所有命令。
```

```sql
EXEC  -- 执行所有事务块内的命令。
```

```sql
MULTI  -- 标记一个事务块的开始。
```

```sql
UNWATCH  -- 取消 WATCH 命令对所有 key 的监视。
```

```sql
WATCH key [key ...]  -- 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
```

## 十一、Redis `脚本`命令

```sql
EVAL script numkeys key [key ...] arg [arg ...]  -- 执行 Lua 脚本。
```

```sql
EVALSHA sha1 numkeys key [key ...] arg [arg ...]  -- 执行 Lua 脚本。
```

```sql
SCRIPT EXISTS script [script ...]  -- 查看指定的脚本是否已经被保存在缓存当中。
```

```sql
SCRIPT FLUSH  -- 从脚本缓存中移除所有脚本。
```

```sql
SCRIPT KILL  -- 杀死当前正在运行的 Lua 脚本。
```

```sql
SCRIPT LOAD script  -- 将脚本 script 添加到脚本缓存中，但并不立即执行这个脚本。
```

## 十二、Redis `服务器`命令

```sql
BGREWRITEAOF  -- 异步执行一个 AOF（AppendOnly File） 文件重写操作
```

```sql
BGSAVE  -- 在后台异步保存当前数据库的数据到磁盘
```

```sql
CLIENT KILL [ip:port][ID client-id]  -- 关闭客户端连接
```

```sql
CLIENT LIST  -- 获取连接到服务器的客户端连接列表
```

```sql
CLIENT GETNAME  -- 获取连接的名称
```

```sql
CLIENT PAUSE timeout  -- 在指定时间内终止运行来自客户端的命令
```

```sql
CLIENT SETNAME connection-name -- 设置当前连接的名称
```

```sql
CLUSTER SLOTS  -- 获取集群节点的映射数组
```

```sql
COMMAND  -- 获取 Redis 命令详情数组
```

```sql
COMMAND COUNT  -- 获取 Redis 命令总数
```

```sql
COMMAND GETKEYS  -- 获取给定命令的所有键
```

```sql
TIME  -- 返回当前服务器时间
```

```sql
COMMAND INFO command-name [command-name ...]  -- 获取指定 Redis 命令描述的数组
```

```sql
CONFIG GET parameter   -- 获取指定配置参数的值
```

```sql
CONFIG REWRITE  -- 对启动 Redis 服务器时所指定的 redis.conf 配置文件进行改写
```

```sql
CONFIG SET parameter value  -- 修改 redis 配置参数，无需重启
```

```sql
CONFIG RESETSTAT  -- 重置 INFO 命令中的某些统计数据
```

```sql
DBSIZE  -- 返回当前数据库的 key 的数量
```

```sql
DEBUG OBJECT key  -- 获取 key 的调试信息
```

```sql
DEBUG SEGFAULT   -- 让 Redis 服务崩溃
```

```sql
FLUSHALL   -- 删除所有数据库的所有key
```

```sql
FLUSHDB   -- 删除当前数据库的所有key
```

```sql
INFO [section]  -- 获取 Redis 服务器的各种信息和统计数值
```

```sql
LASTSAVE  -- 返回最近一次 Redis 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示
```

```sql
MONITOR  -- 实时打印出 Redis 服务器接收到的命令，调试用
```

```sql
ROLE  -- 返回主从实例所属的角色
```

```sql
SAVE   -- 同步保存数据到硬盘
```

```sql
SHUTDOWN [NOSAVE][SAVE]  -- 异步保存数据到硬盘，并关闭服务器
```

```sql
SLAVEOF host port  -- 将当前服务器转变为指定服务器的从属服务器(slave server)
```

```sql
SLOWLOG subcommand [argument]  -- 管理 redis 的慢日志
```

```sql
SYNC  -- 用于复制功能(replication)的内部命令
```

