[TOC]

# redis学习(一) — redis介绍及安装

## **一、redis简介**

       redis是一个高性能的key-value非关系数据库，它可以存键(key)与5种不同类型的值(value)之间的映射(mapping)，支持存储的value类型包括:String(字符串)、list(链表)、set(集合)、zset(有序集合)和hash(散列表)。这些收据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步。

## **二、redis安装**

#### **1、下载最新版的Redis**

打开redis在github上的网站：<https://github.com/MSOpenTech/redis/releases>，选择下载最新版的Redis，写该文字时最新版本是3.2.1版。

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170221001917726-295892063.png)

这里我们选择下载手动安装包Redis-x64-3.2.100.zip。将下载后的安装包解压到磁盘中，最好是没有中文路径，没有特殊字符的目录下，比如：d:\redis目录下，我的解压到D:\Redis_x64_321，解压后可以看到如下文件

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170221235858554-939573732.png)

redis.windows.conf redis的配置文件

redis-benchmark.exe 测试工具，测试redis的读写性能情况

redis-check-aof.exe aof 修复检查日志

redis-check-dump.exe dump 检查数据库文件

redis-cli.exe redis客户端程序

redis-server.exe redis服务器程序

#### **2、添加环境变量**

为了更加方便的使用Redis，可以添加环境变量，在“系统环境变量”中的“Path”变量下添加redis路径，如下所示：

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170222002137054-383911774.png)

确定后启动cmd，运行redis-server测试。

#### **3、启动服务器**

打开一个cmd窗口，使用cd命令切换目录到D:\Redis_x64_321，然后运行：**redis-server.exe redis.windows.conf，**输入之后，会显示如下界面：

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170222004644663-333542763.png)

启动成功后会有一个字符界面，提示连接的端口号是：6379，请不要关闭该服务器，等待客户端连接；这里也可以把redis作成windows服务，不过redis多数情况会在linux平台使用。

#### **4、启动客户端**

再用cmd开启一个命令容器，输入命令：redis-cli -h 127.0.0.1 -p 6379，执行成功后如下所示：

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170222005323866-1144530164.png)

#### 5、**测试并运行**

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170222005621976-1377524235.png)

添加数据：set <key> <value> 

获得数据：get <key>

是否存在：exists <key>

删除数据：del <key>

修改数据：set <key> <value>

帮助命令：help <命令名称>
