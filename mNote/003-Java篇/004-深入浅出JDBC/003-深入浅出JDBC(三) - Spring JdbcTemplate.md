# <font color='#FF6347'>深入浅出JDBC(三) - Spring JdbcTemplate</font>

上一次我们讨论了Dbutils的用法，其实现原理很简单，就是对JDBC的原始操作进行封装。但是无论什么操作，首先得创建Connection或者DataSource对象。在业务项目的开发中，手动地创建和销毁Connection比较繁琐，且不能充分地利用资源。于是有了连接池DBCP和C3P0两个框架的出现，但是业务开发过程中，对连接资源的获取和释放同业务是完全无关的，那能不能就不关心连接的获取和释放，因此Spring将JDBC和IOC结合在一起，可以无感知地对数据库进行操作。

## 1. query与update

无感知操作不代表不需要配置数据库连接相关属性，我们使用xml方式配置基于DBCP的DataSource，然后注入到Spring提供的jdbc操作模板类JdbcTemplate中，这样在业务操作中只要持有这个模板对象即可与数据库交互。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-3.0.xsd">
	
	<!-- DBCP的DataSource对象 -->
	<bean id="basicDataSource" class="org.apache.commons.dbcp.BasicDataSource">
		<property name="driverClassName" value="com.mysql.jdbc.Driver" />
		<property name="url" value="jdbc:mysql://192.168.1.160:3306/lichao" />
		<property name="username" value="wms_dev" />
		<property name="password" value="OL2kfZ4s" />
	</bean>
	
	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
		<property name="dataSource" ref="basicDataSource"></property>
	</bean>
</beans>
```

只要在spring启动时加载这个xml文件，即可获得可以与数据库交互的JdbcTemplate对象。这里我们使用spring junit的方式。

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(locations={"classpath:com/lntea/jdbc/spring/jdbc-template.xml"})
public class JdbcTemplateTest {

	@Autowired
	private JdbcTemplate jdbcTemplate;
	
	@Test
	public void testQueryForObject(){
		// query number of rows
		Integer count = jdbcTemplate.queryForObject("select count(*) from t_wms_goods_stock", Integer.class);
	
```

执行junit测试方法testQueryForObject，通过JdbcTemplate的queryForObject方法，查询商品库存的总记录条数。queryForObject方法，支持一条sql查询，返回指定Class type的结果，也支持sql中?占位符，通过可变数组进行绑定。

```java
// query using a bind variable
Integer countOfWarehouseId = jdbcTemplate.queryForObject("select count(*) from t_wms_goods_stock where warehouse_id = ?", Integer.class, 1L);
System.out.println("countOfWarehouseId:" + countOfWarehouseId);
```

如果需要将查询结果映射成java对象，使用参数RowMapper。

```java
// query and populate a single domain object
GoodsStock gs = jdbcTemplate.queryForObject("select * from t_wms_goods_stock where id = ?",
		new RowMapper<GoodsStock>() {

			public GoodsStock mapRow(ResultSet rs, int rowNum) throws SQLException {
				GoodsStock gs = new GoodsStock();
				gs.setId(rs.getInt("id"));
				gs.setWarehouseId(rs.getLong("warehouse_id"));
				return gs;
			}
		}, 1L);
System.out.println(gs);
```

还可以使用query方法查询一组返回值

```java
// query and populate a number of domain objects
List<GoodsStock> gsList = jdbcTemplate.query("select * from t_wms_goods_stock limit 10",
		new RowMapper<GoodsStock>() {

			public GoodsStock mapRow(ResultSet rs, int rowNum) throws SQLException {
				GoodsStock gs = new GoodsStock();
				gs.setId(rs.getInt("id"));
				gs.setWarehouseId(rs.getLong("warehouse_id"));
				return gs;
			}
		});
System.out.println(gsList);
```

看完查询，我们来试试更新操作，update方法可以极方便地完成

```java
@Test
public void testUpdate(){
	int rowUpdateNum = jdbcTemplate.update("update t_wms_goods_stock set warehouse_id = ? where id = ?", 1L, 1);
	System.out.println("rowUpdateNum:" + rowUpdateNum);
}
```



## 2. 实现原理

那么，JdbcTemplate在底层是如何完成这些简便的api和jdbc的交互呢，其实spring jdbc对jdbc不同的操作做了模块的抽象，来看一个有参数查询的Demo。

```java
// PreparedStatement的创建器，将sql转化成PreparedStatement
PreparedStatementCreator psc = new PreparedStatementCreator() {
	
	public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
		return con.prepareStatement("select count(*) from t_wms_goods_stock where warehouse_id = ?");
	}
};

// PreparedStatement参数组装器，将sql参数设置到PreparedStatement中
PreparedStatementSetter pss = new PreparedStatementSetter() {
	
	public void setValues(PreparedStatement ps) throws SQLException {
		ps.setLong(1, 1L);
	}
};

// 结果解析器，将sql查询结果转换成指定的对象类型
ResultSetExtractor<Integer> rse = new ResultSetExtractor<Integer>() {

	public Integer extractData(ResultSet rs) throws SQLException, DataAccessException {
		while(rs.next()){
			return rs.getInt(1);
		}
		return null;
	}
};

// 执行query操作
int count = jdbcTemplate.query(psc, pss, rse);
System.out.println("count:" + count);
```

spring将jdbc中PreparedStatement操作中可变的部分抽象成三个模块

1. PreparedStatementCreator： PreparedStatement创建器
2. PreparedStatementSetter ：PreparedStatement参数组装器
3. ResultSetExtractor：结果集解析器

如上文中的queryForObject的例子

```java
Integer countOfWarehouseId = jdbcTemplate.queryForObject("select count(*) from t_wms_goods_stock where warehouse_id = ?",
Integer.class, 1L);
```



```java
在实现时，转换成spring自带的上述三个模块的实现类，sql对应SimplePreparedStatementCreator，查询参数对应newArgPreparedStatementSetter，而对结果集的Class类型为Integer的要求封装成getSingleColumnRowMapper，并最终由RowMapperResultSetExtractor完成结果集的转换。

三个模块完成后，又怎么完成jdbc的底层操作呢？query方法调用的是execute方法
```



```java
public <T> T query(
		PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss, final ResultSetExtractor<T> rse)
		throws DataAccessException {
	return execute(psc, new PreparedStatementCallback<T>() {
		@Override
		@Nullable
		public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
			ResultSet rs = null;
			try {
				if (pss != null) {
					pss.setValues(ps);
				}
				rs = ps.executeQuery();
				return rse.extractData(rs);
			}
			finally {
				JdbcUtils.closeResultSet(rs);
				if (pss instanceof ParameterDisposer) {
					((ParameterDisposer) pss).cleanupParameters();
				}
			}
		}
	});
}
```

在execute方法中，创建了PreparedStatementCallback对象，实现doInPreparedStatement方法，封装了PreparedStatement的参数设置，查询执行以及结果提取转换。execute方法的执行就回到了最基础的jdbc操作。

```java
public <T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action)
		throws DataAccessException {
	if (logger.isDebugEnabled()) {
		// 获取sql并输出日志
		String sql = getSql(psc);
		logger.debug("Executing prepared SQL statement" + (sql != null ? " [" + sql + "]" : ""));
	}

	// 获取数据库连接
	Connection con = DataSourceUtils.getConnection(obtainDataSource());
	PreparedStatement ps = null;
	try {
		// 创建PreparedStatement
		ps = psc.createPreparedStatement(con);
		// statement配置
		applyStatementSettings(ps);
		// 执行jdbc查询操作
		T result = action.doInPreparedStatement(ps);
		// 输出warning日志
		handleWarnings(ps);
		return result;
	}
	catch (SQLException ex) {
		// 发生Sql异常，提前释放资源，防止连接池死锁
		// Release Connection early, to avoid potential connection pool deadlock
		// in the case when the exception translator hasn't been initialized yet.
		if (psc instanceof ParameterDisposer) {
			((ParameterDisposer) psc).cleanupParameters();
		}
		String sql = getSql(psc);
		JdbcUtils.closeStatement(ps);
		ps = null;
		DataSourceUtils.releaseConnection(con, getDataSource());
		con = null;
		throw translateException("PreparedStatementCallback", sql, ex);
	}
	finally {
		if (psc instanceof ParameterDisposer) {
			((ParameterDisposer) psc).cleanupParameters();
		}
		JdbcUtils.closeStatement(ps);
		DataSourceUtils.releaseConnection(con, getDataSource());
	}
}
```

对于获取数据库连接，调用的是DataSourceUtils的getConnection方法，对于非事务管理的jdbc操作，起作用的就是一句代码。

```java
Connection con = fetchConnection(dataSource);

private static Connection fetchConnection(DataSource dataSource) throws SQLException {
	Connection con = dataSource.getConnection();
	if (con == null) {
		throw new IllegalStateException("DataSource returned null from getConnection(): " + dataSource);
	}
	return con;
}
```

由DataSource创建Connection连接。而同事务相关的部分，等到以后谈到spring 事务管理时再详细介绍吧。

我们深入的探究了spring JdbcTemplate处理预编译sql的原理，对于普通的Statement操作，实现过程同PreparedStatement相似，这里就不再介绍了，有兴趣自己查看源码。这篇文章中介绍了query和update的例子，delete操作其实雷同于update，而insert操作，因为其涉及到自增主键的返回，单独一章介绍spring的处理方式。