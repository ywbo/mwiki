# <font color='#FF6347'> 深入浅出JDBC(一) - Connection与事务介绍</font>

自从数据需要被持久化存储，程序与数据库之间的交互就是不可避免的操作。而早期程序与不同的数据库的交互方式不同，意味着程序开发不得不面对数据库的具体实现，一旦切换数据库，又是新的学习过程，这样开发人员的效率始终被和数据库的交互所限制。

根据软件设计中的依赖倒置原则，**要针对接口编程，不要针对实现编程**，因而由于数据库的变化而引起实现的变化是违背软件设计的基本原则的。优化的方式是在数据库上提供一层抽象接口，不同数据库根据接口完成自己的实现，而用户使用时，只需面向接口编程，而不用关心内部的实现细节。如果需要更换数据库，只需要要指定不同的数据库标识即可。

Java的JDBC(Java DataBase Connectivity)就是一组这样的抽象API，通过执行SQL语句，为多种关系型数据库提供统一访问。下面以mysql的一个简单查询为例，介绍JDBC的接口和基本构成。

```java
try {
	// 装载驱动
	Class.forName("com.mysql.jdbc.Driver");
} catch (ClassNotFoundException e) {
	e.printStackTrace();
}

// 数据库连接
Connection conn = null;
// sql执行对象
Statement statement = null;
// 结果集
ResultSet resultSet = null;

try {
	// 获取mysql数据库连接
	conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/lichao", "root", "root");
	// 创建sql执行对象
	statement = conn.createStatement();
	// 执行sql，返回结果集
	resultSet = statement.executeQuery("select * from user");
	// 遍历结果集，获取数据
	while(resultSet.next()){
		Long id = resultSet.getLong("id");
		String name = resultSet.getString("name");
		Integer age = resultSet.getInt("age");
		System.out.println("User[id=" + id + ",name=" + name + ",age=" + age + "]");
	}
} catch (SQLException e) {
	e.printStackTrace();
} finally {
	// 关闭各种资源
	if(resultSet!=null)
		try {
			resultSet.close();
		} catch (SQLException e) {
			e.printStackTrace();
		}
	if(statement!=null)
		try {
			resultSet.close();
		} catch (SQLException e) {
			e.printStackTrace();
		}
	if(conn!=null)
		try {
			conn.close();
		} catch (SQLException e) {
			e.printStackTrace();
		}
}
```

1. **Driver**：驱动接口，不同数据库提供自己的驱动实现，用户使用时注册指定的驱动实现。推荐使用Class.forName()的方式注册。
2. **DriverManager**：驱动管理器，用来管理驱动，以及创建连接
3. **Connection**：与数据库的连接会话，用来执行sql并返回结果。使用DriverManager.getConnection()创建新连接。
4. **Statement**：sql执行的具体对象，用户发送简单的sql(不含参数)。子接口PreparedStatement，用于发送含有一个或多个参数的sql语句。
5. **ResultSet**：数据库返回的数据集

## 1. Connection

connection，官方文档的定义是:

> A connection (session) with a specific database. SQL statements are executed and results are returned within the context of a connection.

连接即一个特定数据库的会话，在一个连接的上下文中，sql语句被执行，然后结果被返回。

通过getMetaData方法可以获取数据库中描述的表的信息，支持的sql语法，存储过程以及这个连接的其他性能和能力。

setSchema和setCatalog指定数据库具体的catalog或schema，setReadOnly设置连接是否为可读模式，setTypeMap指定字段类型和java类的映射关系(如果为空集合，会覆盖原有映射map)，setNetworkTimeout则是等待数据库返回的超时时间，setHoldability改变ResultSet在commit之后的Cursor是否关闭。

一个Connection对象中，包含的信息非常之多。在日常开发中，最常接触到的还是事务相关的操作。

## 2. 事务

事务(Transaction)，解决的是多个操作执行时，同时成功或同时失败的问题。比如银行转账时，扣减A的资金和增加B的资金必须一起发生，不然银行就干不下去了。

但是事务的存在，在多用户同时访问相同的数据时会产生冲突。可能出现以下几种不确定情况：

- **更新丢失**

  两个事务同时更新一行数据(事务都未提交)，一个事务对数据的更新把另一个事务对数据的更新覆盖了。

- **脏读**

  一个事务读到了另一个事务未提交的数据操作结果(未提交的数据可能会被回滚)

- **不可重复读**

  一个事务对同一行数据重复读取两次，但是却得到了不同的结果。包括两种情况：

  - 虚读：事务1读取某数据后，事务2对其做了修改，当事务1再次读改数据时得到与前一次不同的值。

  - 幻读：事务在操作过程中进行两次查询，第二次查询的结果包含或缺少了第一次查询中的数据(一般为范围查询，两次查询sql可不一样)。这是因为两次查询过程中有另一事务插入或删除数据造成的。

    

  为了解决上述情况，标准sql规范中，定义了4个事务隔离级别，不同的隔离级别对事务的处理不同。

  1. **Read Uncommitted**(未授权读取)：允许脏读取，但不允许更新丢失。如果一个事务已经开始写数据，则另外一个事务则不允许同时进行写操作，但允许其他事务读此行数据。
  2. **Read Committed**(授权读取)：允许不可重复读取，但不允许脏读取。读取数据的事务允许其他事务继续访问该行数据，但是未提交的写事务将会禁止其他事务访问该行。
  3. **Repeatable Read**(可重复读取)：静止不可重复读取和脏读取，但有时可能出现幻读。读取数据的事务将会禁止写事务(但允许写事务)，写事务则禁止任何其他事务。
  4. **Serializable**(序列化)：提供严格的事务隔离。要求事务序列化执行，不能并发执行。

  隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。

## 3. Connection的事务

JDBC中，一个connection被创建时，默认是auto-commit模式，也即一个sql statement作为一个事务，执行完成后自动commit。如果支持多个statement组成一个事务，则要禁止auto-commit模式。

con.setAutoCommit(false);

Connection支持了标准的四种隔离级别，如果没有设置，则查询数据库的默认隔离级别，也可以用setTransactionIsolation方法手动设置。

```java
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```

Connection使用commit和rollback方法支持事务的提交和回滚。

```java
try {
	conn.setAutoCommit(false);
	// -- do query and update operation
	conn.commit();
} catch (Exception e) {
	conn.rollback();
}
```

同时使用setSavepoint和releaseSavepoint方法支持保存点的设置和释放。

```java
try {
	conn.setAutoCommit(false);
	// -- update 1
	Savepoint s1 = conn.setSavepoint();
	try {
		// -- update2
	} catch (Exception e) {
		conn.releaseSavepoint(s1);
	}
	conn.commit();
} catch (Exception e) {
	conn.rollback();
}
```



## 4. DataSource

DataSource表示一种创建Connection的工厂，在jdk 1.4引入，相对DriverManager的方式更优先推荐使用DataSource。支持三种实现类型：

1. 基本实现：产生一个标准连接对象
2. 连接池实现：将连接对象池化处理，由一个连接池管理中间件支持
3. 分布式事务实现：支持分布式事务，通常也是池化的，由一个事务管理中间件支持。

基于DataSource产生了两个非常常用的数据库连接池框架：DBCP和C3P0，解决了数据库连接的复用问题，极大地提高了数据库连接的使用性能。看一个DBCP的简单用例：

```xml
BasicDataSource basicDataSource = new BasicDataSource();
basicDataSource.setDriverClassName("com.mysql.jdbc.Driver");
basicDataSource.setUrl("jdbc:mysql://localhost:3306/lichao");
basicDataSource.setUsername("root");
basicDataSource.setPassword("root");
// 初始化连接数
basicDataSource.setInitialSize(5);
// 最大连接数
basicDataSource.setMaxActive(30);
// 最大空闲连接数
basicDataSource.setMaxIdle(5);
// 最小空闲连接数
basicDataSource.setMinIdle(2);
Connection conn = basicDataSource.getConnection();
// sql operation
conn.close();
```

关于DBCP和C3P0，具体的这里就不深入了。

JDBC解决了面向不同数据库的统一接口问题，我们只要遵守它的接口编程，则不用关心底层数据库的内部实现。然而程序员对编程便捷的追求是永不止步的。我们不能接受一次简单的查询需要如此多的步骤，打开连接，创建statement对象，解析结果集，最后关闭资源，甚至连JDBC我都不想关心。我的需求很简单，就是提出一个对数据库的请求（请求或更新），返回给我结果，如果是查询，直接映射到java对象最好。我们希望更多地关注真正关心的东西（比如业务），而不是数据的传输过程。于是就出现了各种对JDBC的封装框架，比如apache的dbutils。

------

参考文档：

1. [https://baike.baidu.com/item/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB/2638091](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB%2F2638091)
2. [https://docs.oracle.com/javase/8/docs/api/](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fdocs.oracle.com%2Fjavase%2F8%2Fdocs%2Fapi%2F)
3. [https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fdocs.oracle.com%2Fjavase%2Ftutorial%2Fjdbc%2Fbasics%2Ftransactions.html)