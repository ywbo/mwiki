https://github.com/Snailclimb/JavaGuide/blob/main/docs/database/mysql/mysql-questions-01.md
MySQL 面试必看
关系型数据库（例如：MySQL、SQL Server、Oracle 等）事务都有 ACID 特性：
- `原子性（Atomicity）` ： 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
- `隔离性（Isolation）`： 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
- `持久性（Durabilily）`： 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。
- `一致性（Consistency）`： 执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；

> #### 🌈 这里要额外补充一点：只有保证了事务的持久性、原子性、隔离性之后，一致性才能得到保障。也就是说 A、I、D 是手段，C 是目的！

