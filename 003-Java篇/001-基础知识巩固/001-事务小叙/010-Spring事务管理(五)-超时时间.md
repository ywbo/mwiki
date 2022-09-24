# <font color='#FF6347'> Spring事务管理(五)-超时时间</font>

> 问：为什么会写关于超时间的这篇文章。
>
> 答：**mybatis3.4.0以下的版本存在@Transactional的timeout无效的情况（使用JdbcTemplate手动执行是有效的），3.4.0及以上的版本修复了此问题。**

关于Spring事务超时时间的实现，一直都没太弄清楚，终于在看到一篇[事务超时](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.heartthinkdo.com%2F%3Fp%3D910)文章后，通过测试用例证明**通常情况下@Transactional中配置的timeout都是无效的。**

首先说明下测试的注意事项，就是除了@Transactional的timeout配置外，不要配置其他超时时间，比如mybatis xml中sql的timeout，jdbc properties中的socket timeout(connectionTimeout和socketTimeout)以及mysql的innodb_lock_wait_timeout(默认50s)。

测试方法如下，超时时间为1s，线程中等待3s，使用Mybatis方式配置sql

```java
@Transactional(timeout=1)
public void batchUpdate(){
    
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    
    int updateRow = userMapper.batchUpdate(userList);
    System.out.println("updateRow:" + updateRow);
}
```

测试结果，事务执行成功了，第一次测试的时候，我也是惊呆了。什么鬼？？项目里配置的timeout竟然都是摆设。

再来测试JdbcTemplate直接执行sql的方法

```java
public class UserJdbcTemplateMapper {

	private JdbcTemplate jdbcTemplate;
	
	public int updateUser(){
		return jdbcTemplate.update("update user set age = 10 where id = 1");
	}
}
```

@Transactional还是上面的配置，测试结果为：

```java
org.springframework.transaction.TransactionTimedOutException: Transaction timed out: deadline was Fri Feb 02 20:48:43 CST 2018
```

建议大家自己测试一下，亲自体验的感觉特别好。下面根据DataSourceTransactionManager来分析其原理。

抽象AbstractPlatformTransactionManager对@Transactional的超时时间没有任何处理，而在DataSourceTransactionManager的doBegin方法中将其设置到ConnectionHolder中。

```java
if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
    txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
}
```

实际上就是设置了一个deadline

```java
public void setTimeoutInSeconds(int seconds) {
    setTimeoutInMillis(seconds * 1000);
}

public void setTimeoutInMillis(long millis) {
    this.deadline = new Date(System.currentTimeMillis() + millis);
}
```

而对deadline的校验的方法就是checkTransactionTimeout，在获取超时时间的方法里被执行

```java
// 获取剩余时间(单位为秒)
public int getTimeToLiveInSeconds() {
    double diff = ((double) getTimeToLiveInMillis()) / 1000;
    int secs = (int) Math.ceil(diff);
    checkTransactionTimeout(secs <= 0);
    return secs;
}

// 获取剩余时间(单位为毫秒)
public long getTimeToLiveInMillis() throws TransactionTimedOutException{
    if (this.deadline == null) {
        throw new IllegalStateException("No timeout specified for this resource holder");
    }
    long timeToLive = this.deadline.getTime() - System.currentTimeMillis();
    checkTransactionTimeout(timeToLive <= 0);
    return timeToLive;
}

// 校验是否超时，抛出TransactionTimedOutException异常
private void checkTransactionTimeout(boolean deadlineReached) throws TransactionTimedOutException {
    if (deadlineReached) {
        setRollbackOnly();
        throw new TransactionTimedOutException("Transaction timed out: deadline was " + this.deadline);
    }
}
```

而对获取剩余时间方法的调用为DataSourceUtils的applyTimeout，将超时时间转换为JDBC的Statement的queryTimeout，因而Spring事务的超时时间也就是通过Statement的超时来实现。

```java
public static void applyTimeout(Statement stmt, @Nullable DataSource dataSource, int timeout) throws SQLException {
    Assert.notNull(stmt, "No Statement specified");
    ConnectionHolder holder = null;
    if (dataSource != null) {
        holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    }
    if (holder != null && holder.hasTimeout()) {
        // Remaining transaction timeout overrides specified value.
        stmt.setQueryTimeout(holder.getTimeToLiveInSeconds());
    }
    else if (timeout >= 0) {
        // No current transaction timeout -> apply specified value.
        stmt.setQueryTimeout(timeout);
    }
}
```

可是applyTimeout的调用者只有JdbcTemplate和TransactionAwareDataSourceProxy。在JdbcTemplate的execute方法中通过applyStatementSettings方法设置了超时时间。而TransactionAwareDataSourceProxy则是JDK代理的InvocationHandler的实现类，感觉应该是DataSource的代理类。但是纵观Spring事务管理的核心实现方法中获取DataSource的操作，都没有对原始DataSource进行代理的操作，甚至在DataSourceTransactionManager的setDataSource方法中，判断如果DataSource为TransactionAwareDataSourceProxy类型，则获取其原始DataSource。

```java
public void setDataSource(@Nullable DataSource dataSource) {
    if (dataSource instanceof TransactionAwareDataSourceProxy) {
        this.dataSource = ((TransactionAwareDataSourceProxy) dataSource).getTargetDataSource();
    }
    else {
        this.dataSource = dataSource;
    }
}
```

因此就算在xml中配置了TransactionAwareDataSourceProxy来代理原始DataSource，对于DataSourceTransactionManager来说，只是虚设。

```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy">
    <constructor-arg index="0" ref="basicDataSource"></constructor-arg>
</bean>
```

因此对于Spring事务超时时间的设置，要格外的注意，可以参考[事务超时](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fwww.heartthinkdo.com%2F%3Fp%3D910)这篇文章，对各层的超时时间的介绍及作用相当清楚。