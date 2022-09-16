## Redis

### 1.NoSql数据库

nosql数据库是菲关系型数据库，是Not Only Sql，意思是不仅仅只有SQL；

Nosql的四大分类：

1.K-V键值对： 新浪-redis,美团-reids+Tair

2.文档型数据库（bson格式和json一样）：

    MongoDB:
        MongoDB是一个基于分布式文件存储的数据库，C++编写，主要用来处理大量的文档！
        MongoDB是一个介于关系型和非关系型数据之间的中间产品！MongoDB是非关系型数据库中功能最丰富，最像关系型数据库的！
    
    ConthDB:

3.列存储数据库

    HBase
    
    分布式文件系统

4.图关系数据库：他不是存图形的，存放的是关系，比如：朋友圈社交网络，广告推荐；

### 2.Redis

Redis是（Remote Dictionary Server），远程字典服务；它是一个开源的使用ANSI C语言编写，支持网络，可基于内存亦可持久化的日志型，Key-Value 数据库，并提供多种语言的API；

redis常用命令：

    select 3 -- 切换数据库到3上（redis有16个数据库）； 
    dbsize -- 查看数据库大小； 
    keys * -- 查看当前数据库下所有的KEY；
    flushall -- 清空全部DB内容； 
    flushdb-- 清空当前DB

为什么redis的端口号是6379？

因为一个意大利娘们MERZ的名字在手机9键上输入键是6379，MERZ长期以来被redis的作者及其朋友当作愚蠢的代名词

 **redis为啥是单线程的？** 

Redis是单线程的，他是很快的，官方表示，redis是基于内存操作的，cpu不是redis性能的瓶颈，cpu是多线程性能的瓶颈，Redis的瓶颈是由所在机器的内存和网络带宽决定，所以单线程就可以满足redis的实现，不需要使用多线程。

 **redis为什么单线程还很快？** 

首先，高性能的服务器不一定都是多线程的，多线程在单个服务器上（cpu上下文切换，这是一个耗时的操作）不一定比单线程效率高；redis是将全部数据放在内存中的避免了cpu上下文切换，所以说使用单线程去操作效率就是最高的，多线程效率的瓶颈是cpu，而cup上线文会切换，非常耗时。对于内存系统来说，如果没有上下文切换效率就是最高的！多次读写都是在一个cpu上的，在内存情况下，这个就是最佳的方案。


### 2.1 Redis的数据类型：

 **五种基本数据类型： Strings(字符串),Lists（列表）,Hashes(散列),Set（集合）,Zset（有序集合）; **

 **2.1.1 String类型** ：

    set key1 v1 -- 设置值；
    get key1 -- 获取值；
    key * -- 获取所有Key;
    EXISTS key1 -- 判断某一个key是否存在；
    APPEND key1 "hello"  -- 追加字符串，如果当前key不存在，就相当于与setkey;
    STRLEN key1 -- 获取字符串的长度；
    incr views  -- 设置key为views键的值自增；
    decr views  -- 设置key为view键的值自减；
    incrby views 10  -- 设置key为views的步长为10，指定增量是10；
    setex (set with expire)  -- 设置过期时间；
    setnx (set if not exist) -- 不存在在设置（在分布式锁中会常常使用）

String类型的使用场景：value除了是我们的字符串还可以是我们的数字！ 可以做计数器（粉丝关注，点击量），统计多单位的数量；
    
 **2.1.2 List(列表)类型** ：实际上是一个链表，可以通过left，right插入值；它可以实现消息队列！ 消息队列（Lpush Rpop）,栈（Lpush Lpop）

 **2.1.3 Set（集合）类型** ：可以通过集合的并集，交集，差集，来实现微博的共同关注，共同好友；等到互关用户；二度好友；

 **2.1.4 Hash（散列）类型** ：Map集合，适合存储经常变更的信息user name age；数据结构为： key - map(key ,value);例：hset myhash(key) zhaomeng niubi 。hash更适合存储对象，String更适合字符串存储；

 **2.1.5 Zset(有序集合)类型**  ：在set的基础上增加了一个值（score），用来排序; 例： zadd myset 1 one ; zadd myset 2 two 3 three ; 自动按1-3排序；可通过Zset实现热门搜榜，成绩表等。zset也可以带上权重；

 **三种特殊的数据类型：Geospatial(地理位置)；Hyperloglog(基数统计)；Bitmap（位图）；** 

Geospatial(地理位置)：redis在3.2版本已经推出了，这个功能可以推理出地理信息；用他可以实现两点之间的距离，半径范围内的人（附近的人），朋友的定位；

    # geoadd 添加地理位置
    # 规则：两极无法添加，有效经度为-180-180；有效纬度为-85.0511-85.0511；一般都是下载城市经纬度通过Java程序导入redis再用；
    例：geoadd china:city 116.40 39.90 beijing
        geoadd china:city 121.47 31.23 shanghai
    
    # geopos 从key里返回所有给定位置元素的位置（经纬度）；
    # 例：geopos china:city beijing;
       返回：“116.3999”
            “39.9”
    
    # geodist 两点之间的距离，
    # 例：geodist china:city beijing shanghai km
    # 返回：“1068.3788”
    # 单位： m 表示单位为 米； km 千米； mi 单位为英里； ft单位为英尺；
    
    # georadius china:city 110 30 1000 km 找经纬度为110,30的周边1000公里的城市；

geo 底层的实现原理其实就是Zset!，我们可以使用Zset命令来操作geo； 查看完附近的人时候不想保留自己的位置信息可以用Zset的 zrem china:city beijing 命令把自己位置信息清除；

Hyperloglog(基数统计)： 常用于做基数统计，redis2.8.9版本就更新了Hyperloglog数据结构；

优点：占用固定内存，2^64不同的元素的技术，只需耗费12KB的内存！如果从内存角度讲Hyperloglog是首选！

网页的UV(用户访问量)的实现，传统的方式，set保存用户的id（set类型自动去重，保证了一个用户多次访问后还是1），然后就可以统计set中的元素数量作为标准判断！但这种方式保存了大量的用户Id，就会变的麻烦。我们的目的是计数而不是保存Id；

用Hyperloglog来实现的话只需缓存每个用户，最后用PFMERGE合并，PFCOUNT 拿到计数值就行。
    
    常用命令：PFadd mykey a b c d ..  # 创建第一组元素 mykey
             PFCOUNT mykey  # 统计 mykey 元素的基数
             PFMERGE mykey mykey2 # 合并两个key 

Bitmap（位图）：位存储，常用于统计用户信息，活跃，不活跃；登录，未登录；365打卡；这种两个状态的都可以使用Bitmaps,bitmaps 数据结构是位图，都是操作二进制位来进行记录，就只有0和1两个状态！

365天的打卡信息=365bit; 1个字节 = 8个bit;只需要45个字节就能存储一个用户365天的打卡情况。

    # 例 使用bitmaps 来收集打卡的天数；
    # 周一：1 周二：0 周三 ：0 周四：1
     setbit sign 0 1;   setbit sign 1 0;   setbit sign 2 0;...
    # 查看哪个是否打卡；
     getbit sign 2  返回：0；
    # 统计打卡天数
     bitcount sign 

### 3 Redis的事务：

事务的本质是一组命令的集合！一个事务中的所有命令都会被序列化，在事务执行过程中，会按照顺序执行；

事务的特性： 一次性，顺序性，排他性，执行一系列的命令！

Redis单条命令是保证原子性的，同时成功，同时失败；但redis的事务不保证原子性；redis事务没有隔离级别的概念！只有在发起执行命令的时候才会执行！

redis事务：
    1. 开启事务（multi）；
    2. 命令入队；
    3. 执行事务(exceut)；
    4. 放弃事务（DISCARD）；

redis事务的异常: 1. 若redis的命令输入有错，则类似于java中的编译型异常， **事务中所有命令都不会被执行** ；2.如果事务队列中存在语法性错误，则相当于java中的运行期异常，但由于redis的事务不保证原子性，所以， **它只会将有语法错误的那条命令执行失败，抛出异常，其他命令依旧能执行成功！** 

### 4 Redis的持久化：

redis的持久化有两种方案，RDB,AOF;由于Redis是内存级数据库，如果不将内存中的数据库状态保存在磁盘上，一旦服务器进程退出，数据库状态也会消失，断电即失，所以必须持久化；

 **RDB** ：在指定的时间内，将内存中的数据集快照写入到磁盘中，也就是生成Snapshot快照，它恢复时是将快照文件直接读到内存中；

持久化过程：redis 会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程结束了，再用这个临时文件替换上一次持久化好的文件。整个过程中，主进程是不进行任何IO操作的；这就保证了极高的性能。如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那么RDB方式就要比AOF方式更加高效。RDB的缺点是最后一次持久化后的数据可能丢失。我们默认的就是RDB，一般情况下不需要修改这个配置；

RDB保存的文件是dump.rdb 都是在我们的配置文件中快照进行配置的！ save 60 50 只要60s内修改了5次key，就会触发rdb操作；

触发机制：1. save 的规则满足的情况下，会自动触发rdb规则；2.执行flushall命令，也会触发我们的rdb规则！3.退出redis,也会产生rdb文件！

RDB文件的恢复：只需要将dump.rdb文件放入redis的启动目录下（usr/local/bin），redis启动的时候就会自动检查dump.rdb恢复其中的数据；有时候生产环境会将这文件备份；

RDB的优缺点：

    优点：1.适合大规模的数据恢复！
          2.对数据的完整性要求不高！
    缺点：1.需要一定的事件间隔，如果redis意外宕机了，这个最后一次持久化到宕机之间的数据就无法恢复；
         2.fork进程的时候，会占用一定的内存空间。

![输入图片说明](https://img-blog.csdnimg.cn/20210602190336852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0OTM5MzA4,size_16,color_FFFFFF,t_70)

 **AOF：** aof 是以日志的形式记录每个写操作，将redis执行过的所有指令记录下来（读操作不记录），只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据，还言之，redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作；

aof保存的是appendonly.aof文件；配置文件中默认是不开启的，需要手动配置（将appendonly no -> appendonly yes）；若appendonly.aof文件有错误，redis则无法启动，此时我们需要修复该文件，此时redis为我们提供了一个工具redis-check-aof通过命令 redis-check-aof --fix appendonly.aof 修复appendonly.aof文件，重启即可恢复；aof默认就是文件的无限制追加，但当文件达到64M的时候，就会fork一个新的进程重写aof文件；

aof的优缺点：

    优点：
    1.每一次修改都同步，文件的完整性会更好；
    2.每秒同步一次，可能会丢失一秒的数据；
    缺点：
    1. 相对于数据文件来说，aof远远大于rdb，修复速度也比rdb慢！
    2.Aof运行效率也要比rdb慢，所以我们redis默认的配置就是rdb持久化；

### 5 Redis发布/订阅：

redis发布/订阅（pub/sub）是一种消息通信模式：发送者（pub）发送消息，订阅者（sub）接收消息。微信，微博，关注系统！Redis客户端可以订阅任何数量的频道；

命令：
    
    # 1.PSUBSCRIBE 频道名 [频道名...]  ：订阅一个或多个符合给定模式的频道；
    # 2.PUBLISH 频道名 消息内容   ： 将消息发送到指定的频道；   （消息的发送者）
    # 3.SUBSCRIBE 频道名 [频道名...]  ：订阅给定的一个或多个频道信息；  （消息的订阅者）

pub/sub 从字面意思就是发布（publish）与订阅（Subscribe）;在Redis中，你可以设定某一个key值进行消息发布及消息订阅，当一个key值进行了消息发布后，所有订阅它的客户端都会收到相应的消息，这一功能最明显的用法就是实时消息比如即时聊天，每个论坛上的官方消息推送，（其实就是相当于你登录论坛之后，就默认订阅了这个论坛的频道）
