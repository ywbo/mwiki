# <font color='#FF6347'>深入浅出JDBC(二) - Dbutils</font>

对JDBC的封装，主要在三个方面的优化：

1. 资源的开启和关闭，每次与数据库的交互都写一遍，造成大量的重复
2. sql对象和参数传递的简化
3. 结果集与java对象的映射

Dbutils作为一种初级的JDBC封装框架，对JDBC进行了一定的封装，但是面对复杂业务的支持则不够。来看一个简单的demo：

先定义个Connection的帮助类，用来注册驱动和创建连接

```java
public class ConnectionUtil {
	
	static {
		try {
			Class.forName("com.mysql.jdbc.Driver");
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}

	public static Connection getConnection() throws SQLException{
		return DriverManager.getConnection("jdbc:mysql://192.168.1.160:3306/lichao", "root", "root");
	}
}
```

Dbutils以QueryRunner为入口，传入Connection连接和sql，以及结果集的处理器

```java
ResultSetHandler<User> rsh = new ResultSetHandler<User>() {

	@Override
	public User handle(ResultSet rs) throws SQLException {
		rs.next();
		Long id = rs.getLong("id");
		String name = rs.getString("name");
		Integer age = rs.getInt("age");
		return new User(id, name, age);
	}
};

QueryRunner queryRunner = new QueryRunner();
User user = queryRunner.query(ConnectionUtil.getConnection(), "select * from user limit 1", rsh);
System.out.println(user);
```

我们需要创建一个ResultSetHandler的实现类，并实现handler方法，方法中完成ResultSet到java对象之间的转换。

Connection帮助类减少了Connection创建的重复代码，QueryRunner帮我们处理了Statement的操作以及资源的关闭(Connection默认是不关闭的)，但是ResultSet到Java对象的映射还是由自己来实现。可不可以将结果集和java对象的映射再简化呢？Dbutils提供了ResultSetHandler的一种实现BeanHandler，支持将数据库字段直接映射到java对象中。

```java
ResultSetHandler<User> h = new BeanHandler<User>(User.class);
QueryRunner queryRunner = new QueryRunner();
User user = queryRunner.query(ConnectionUtil.getConnection(), "select * from user limit 1", h);
System.out.println(user);
```

BeanHandler把数据库字段同User类中的属性进行匹配，反射生成新的对象。

但是存在一个问题，数据库字段一般以下划线分隔，而Java则是驼峰式命名，因此字段名称直接匹配往往不行。

Dbutils提供了BeanProcessor来处理这个问题，只要将数据库字段名和java类中的属性名通过Map传入，即可完成映射关系。比如增加一个字段月薪(month_salary，对应属性monthSalary)。

```java
Map<String, String> map = new HashMap<String, String>();
map.put("month_salary", "monthSalary");
BeanProcessor customBeanProcessor = new BeanProcessor(map);
ResultSetHandler<User> h = new BeanHandler<User>(User.class,new BasicRowProcessor(customBeanProcessor));

QueryRunner queryRunner = new QueryRunner();
User user = queryRunner.query(ConnectionUtil.getConnection(), "select * from user limit 1", h);
System.out.println(user);
```

Dbutils对查询多条数据也提供了支持

```java
ResultSetHandler<List<User>> h = new BeanListHandler<User>(User.class);
QueryRunner queryRunner = new QueryRunner();
List<User> users = queryRunner.query(ConnectionUtil.getConnection(), "select * from user", h);
for(User user : users){
	System.out.println(user);
}
```

Dbutils还提供了异步处理的方式，AsyncQueryRunner类支持了这个特性。

```java
ResultSetHandler<List<User>> rsh = new BeanListHandler<User>(User.class);
AsyncQueryRunner asyncQueryRunner = new AsyncQueryRunner(Executors.newCachedThreadPool());

Future<List<User>> future = asyncQueryRunner.query(ConnectionUtil.getConnection(),  "select * from user", rsh);

List<User> users = null;
try {
	users = future.get(5, TimeUnit.SECONDS);
} catch (InterruptedException | ExecutionException | TimeoutException e) {
	e.printStackTrace();
}

for(int i=0;i<users.size();i++){
	System.out.println(users.get(i));
}
```

Dbutils也支持使用DataSource获取数据库连接

```java
ResultSetHandler<User> rsh = new ResultSetHandler<User>() {

	@Override
	public User handle(ResultSet rs) throws SQLException {
		rs.next();
		Long id = rs.getLong("id");
		String name = rs.getString("name");
		Integer age = rs.getInt("age");
		return new User(id, name, age);
	}
};

DataSource dataSource = null;// establish DataSource
QueryRunner queryRunner = new QueryRunner(dataSource);
User user = queryRunner.query("select * from user limit 1", rsh);
System.out.println(user);
```

Dbutils对JDBC的封装避免了冗杂的代码，并提供了多种方式的API，支持Connection和DataSource两种方式获得连接。在简单的项目中，使用Dbutils会带来很大的便利。

------

参考文档：

1. [http://commons.apache.org/proper/commons-dbutils/examples.html](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fcommons.apache.org%2Fproper%2Fcommons-dbutils%2Fexamples.html)