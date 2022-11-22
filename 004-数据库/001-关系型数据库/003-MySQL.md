## 关系型数据库（例如：MySQL、SQL Server、Oracle 等）事务都有 ACID 特性：
- `原子性（Atomicity）` ： 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
- `隔离性（Isolation）`： 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
- `持久性（Durabilily）`： 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。
- `一致性（Consistency）`： 执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；

> #### 🌈 这里要额外补充一点：只有保证了事务的持久性、原子性、隔离性之后，一致性才能得到保障。也就是说 A、I、D 是手段，C 是目的！

## 索引

### 建立索引条件：

1. 不为 NULL 的字段；
2. 被频繁查询的字段；
3. 被作为条件查询的字段；
4. 频繁需要排序的字段；
5. 被经常频繁用于连接的字段；
6. 被频繁更新的字段应该慎重建立索引；
7. 考虑在字符串类型的字段上使用 [前缀索引](https://www.cnblogs.com/studyzy/p/4310653.html) 代替普通索引；
8. 尽可能的考虑建立联合索引而不是单列索引；
9. 注意避免冗余索引；
10. 避免索引失效；

### 创建索引
在执行CREATE TABLE语句时可以创建索引，也可以单独用CREATE INDEX或ALTER TABLE来为表增加索引。

1．ALTER TABLE
ALTER TABLE用来创建普通索引、UNIQUE索引或PRIMARY KEY索引。
```sql
ALTER TABLE table_name ADD INDEX index_name (column_list)

ALTER TABLE table_name ADD UNIQUE (column_list)

ALTER TABLE table_name ADD PRIMARY KEY (column_list)
```

其中table_name是要增加索引的表名，column_list指出对哪些列进行索引，多列时各列之间用逗号分隔。索引名index_name可选，缺省时，MySQL将根据第一个索引列赋一个名称。另外，ALTER TABLE允许在单个语句中更改多个表，因此可以在同时创建多个索引。

2．CREATE INDEX
CREATE INDEX可对表增加普通索引或UNIQUE索引。
```sql
CREATE INDEX index_name ON table_name (column_list)

CREATE UNIQUE INDEX index_name ON table_name (column_list)
```
table_name、index_name和column_list具有与ALTER TABLE语句中相同的含义，索引名不可选。另外，不能用CREATE INDEX语句创建PRIMARY KEY索引。

3．索引类型
在创建索引时，可以规定索引能否包含重复值。如果不包含，则索引应该创建为PRIMARY KEY或UNIQUE索引。对于单列惟一性索引，这保证单列不包含重复的值。对于多列惟一性索引，保证多个值的组合不重复。

PRIMARY KEY索引和UNIQUE索引非常类似。事实上，PRIMARY KEY索引仅是一个具有名称PRIMARY的UNIQUE索引。这表示一个表只能包含一个PRIMARY KEY，因为一个表中不可能具有两个同名的索引。

下面的SQL语句对students表在sid上添加PRIMARY KEY索引。
```sql
ALTER TABLE students ADD PRIMARY KEY (sid)
```
### 删除索引
可利用ALTER TABLE或DROP INDEX语句来删除索引。类似于CREATE INDEX语句，DROP INDEX可以在ALTER TABLE内部作为一条语句处理，语法如下。
```sql
DROP INDEX index_name ON talbe_name

ALTER TABLE table_name DROP INDEX index_name

ALTER TABLE table_name DROP PRIMARY KEY
```
其中，前两条语句是等价的，删除掉table_name中的索引index_name。

第3条语句只在删除PRIMARY KEY索引时使用，因为一个表只可能有一个PRIMARY KEY索引，因此不需要指定索引名。如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引。

如果从表中删除了某列，则索引会受到影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。

[MySQL高频面试考点2 - 关于索引的那些事](https://leetcode.cn/circle/discuss/N5PqWI/)

### 索引失效

[Mysql索引查询失效的情况](https://www.cnblogs.com/wdss/p/11186411.html)

------

## 为何使用 B+ 树而非二叉查找树做索引

B+树：

- 1）B+ 树本质是一棵查找树，自然支持范围查询和排序。

- 2）在符合某些条件（聚簇索引、[覆盖索引](https://so.csdn.net/so/search?q=覆盖索引&spm=1001.2101.3001.7020)等）时候可以只通过索引完成查询，不需要回表。

- 3）查询效率比较稳定，因为每次查询都是从根节点到叶子节点，且为树的高度。

我们知道**二叉树的查找**效率为 O(logn)，当树过高时，查找效率会下降。另外由于我们的索引文件并不小，所以是存储在磁盘上的。

文件系统需要从磁盘读取数据时，一般以页为单位进行读取，假设一个页内的数据过少，那么操作系统就需要读取更多的页，涉及磁盘随机 I/O 访问的次数就更多。将数据从磁盘读入内存涉及随机 I/O 的访问，是数据库里面成本最高的操作之一。

因而这种树高会随数据量增多急剧增加，每次更新数据又需要通过左旋和右旋维护平衡的二叉树，不太适合用于存储在磁盘上的索引文件。

## 为何使用 B+ 树而非 B 树做索引？

在此之前，先来了解一下 B+ 树和 B 树的区别：

> - **B 树**：非叶子结点和叶子结点都存储数据，因此查询数据时，时间复杂度最好为 O(1)，最坏为 O(log n)。而 B+ 树只在叶子结点存储数据，非叶子结点存储关键字，且不同非叶子结点的关键字可能重复，因此查询数据时，时间复杂度固定为 O(log n)。
>
> - **B+ 树**：叶子结点之间用链表相互连接，因而只需扫描叶子结点的链表就可以完成一次遍历操作，B 树只能通过中序遍历。

### 那么为什么 B+ 树比 B 树更适合应用于数据库索引？

B+ 树减少了 IO 次数。
由于索引文件很大因此索引文件存储在磁盘上，B+ 树的非叶子结点只存关键字不存数据，因而单个页可以存储更多的关键字，即一次性读入内存的需要查找的关键字也就越多，磁盘的随机 I/O 读取次数相对就减少了。

B+ 树查询效率更稳定
由于数据只存在在叶子结点上，所以查找效率固定为 O(log n)，所以 B+ 树的查询效率相比B树更加稳定。

B+ 树更加适合范围查找
B+ 树叶子结点之间用链表有序连接，所以扫描全部数据只需扫描一遍叶子结点，利于扫库和范围查询；B 树由于非叶子结点也存数据，所以只能通过中序遍历按序来扫。也就是说，对于范围查询和有序遍历而言，B+ 树的效率更高。

[MySQL 高频面试题 - 为什么 B+ 树比 B 树更适合应用于数据库索引？](https://leetcode.cn/circle/discuss/F7bKlM/)

------

## MySQL 事务

- 事务的特性

- 并发事务带来的问题

  - 脏读
  - 不可重复读
  - 幻读

- 事务的隔离级别

  ![img](https://img-blog.csdnimg.cn/2019052019551758.png)

- MySQL 事务与 Oracle 事务区别



------

## MySQL 锁

1. 按照锁的`粒度`来划分：表级锁，行级锁

   - **表级锁**：是粒度最大的一种锁，表示对当前操作的整张表加锁，它实现简单，资源消耗较少，被大部分MySQL引擎支持。
   - **行级锁**：是锁定粒度最细的一种锁，表示只针对当前操作的行进行加锁。行级锁能大大减少数据库操作的冲突。其加锁粒度最小，但加锁的开销也最大。

2. 按照锁的`性质`来划分：共享锁，排它锁

   - **共享锁（Share Lock）**：Share Lock，S 锁，又称**读锁**，用于所有的只读数据操作。
     - S 锁并非独占，允许多个并发事务对同一资源加锁，但加 S 锁的同时不允许加 X 锁，即资源不能被修改。S 锁通常读取结束后立即释放，无需等待事务结束。
   - **排它锁（Exclusive Lock）**：X 锁，又称**写锁**，表示对数据进行写操作。
     - X 锁仅允许一个事务对同一资源加锁，且直到事务结束才释放，其他任何事务必须等到 X 锁被释放才能对该页进行访问。

3. 按照`主观`意识来划分：乐观锁，悲观锁

   - **乐观锁（Optimistic Lock）**：顾名思义，从主观上认定资源是不会被修改的，所以不加锁读取数据，仅当更新时用版本号机制等确认资源是否被修改。
     - 乐观锁适用于多读的应用类型，可以系统提高吞吐量。
   - **悲观锁（Pessimistic Lock）**：正如其名，具有强烈的独占和排它特性，每次读取数据时都会认为会被其它事务修改，所以每次操作都需要加上锁。

4. 什么是MVCC以及如何实现？

   MVCC 的英文全称是 Multiversion Concurrency Control，中文意思是多版本并发控制，可以做到读写互相不阻塞，主要用于解决不可重复读和幻读问题时提高并发效率。

   其原理是通过数据行的多个版本管理来实现数据库的并发控制，简单来说就是保存数据的历史版本。可以通过比较版本号决定数据是否显示出来。读取数据的时候不需要加锁可以保证事务的隔离效果。

   - [MySQL的MVCC实现原理](https://blog.csdn.net/qq_40634846/article/details/123554485)
   - [MySQL的MVCC详解](https://www.cnblogs.com/xuwc/p/13873611.html)

5. 隔离级别与锁的关系

   - 在 Read Uncommitted 级别下，读取数据不需要加共享锁，这样就不会跟被修改的数据上的排他锁冲突；

   - 在 Read Committed 级别下，读操作需要加共享锁，但是在语句执行完以后释放共享锁；

   - 在 Repeatable Read 级别下，读操作需要加共享锁，但是在事务提交之前并不释放共享锁，也就是必须等待事务执行完毕以后才释放共享锁；

   - 在 SERIALIZABLE 级别下，限制性最强，因为该级别锁定整个范围的键，并一直持有锁，直到事务完成。

------

[MySQL面试集](https://blog.csdn.net/adminpd/article/details/122910606)

