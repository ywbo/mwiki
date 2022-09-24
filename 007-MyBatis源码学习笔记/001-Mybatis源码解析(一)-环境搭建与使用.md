[TOC]

# Mybatis源码解析(一)-环境搭建与使用

最近开始重看mybatis源码，借此机会写一个源码解析的系列文章，以解决看完源码脑袋空空、想说又无从说起的困境。这个系列主要针对mybatis的配置解析及执行流程、以及mybatis和spring的整合过程，侧重于执行原理的分析和梳理，不会对某个类或方法详尽地介绍。这一章，作为开篇，介绍mybatis的环境搭建和基本用法。

先说下本次使用源码的版本：

mybatis版本为3.4.6

spring使用5.0.0.RELEASE版本

关于mybatis的用法可以参考[官方文档](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fmybatis.org%2Fmybatis-3%2Fzh%2Findex.html)，常用的基本上都列举了，可以很容易上手。对mybatis不太了解的，也可以通览下整个文档，做一个详细的学习。

## 一、搭建MyBatis使用环境

下面以user表的查询和更新举例，搭建一套简单的mybatis使用环境，测试和研究mybatis使用原理。

建一个user表

```sql
-- ----------------------------
-- Table structure for t_user
-- ----------------------------
DROP TABLE IF EXISTS `t_user`;
CREATE TABLE `t_user`  (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT 'id',
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_as_ci NULL DEFAULT NULL COMMENT '姓名',
  `age` int NULL DEFAULT NULL COMMENT '年龄',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_as_ci ROW_FORMAT = Dynamic;

```

创建 User实体对象，这里使用Lombok注解

```java
@Data
public class User {

	private Long id;
	@NonNull
	private String name;
	@NonNull
	private Integer age;
}
```

再创建UserMapper接口，@Repository注解是为了整合spring使用。

```java
@Repository
public interface UserMapper {

	User getUser(Long id);
	
	int insert(User user);
	
	int delete(Long id);
}
```

mybatis配置文件mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration 
	PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
	"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
	<typeAliases>
		<package name="com.lcifn.orm.mybatis.bean" />
	</typeAliases>
	<environments default="development">
		<environment id="development">
			<transactionManager type="JDBC"></transactionManager>
			<dataSource type="POOLED">
				<property name="driver" value="com.mysql.jdbc.Driver" />
				<property name="url" value="jdbc:mysql://ip:3306/user" />
				<property name="username" value="admin" />
				<property name="password" value="admin" />
			</dataSource>
		</environment>
	</environments>
	<mappers>
		<package name="com.lcifn.orm.mybatis.mapper.test"/>
	</mappers>
</configuration>
```

写一个测试类UserTest，获取id=1的user信息

mybatis执行流程如下：

1. SqlSessionFactoryBuilder构建SqlSessionFactory工厂类，目的是读取mybatis配置文件到Configuration对象中。
2. SqlSessionFactory产生SqlSession，目的是构建和封装事务对象和执行器对象。
3. SqlSession获取Mapper接口，生成Mapper接口的代理实现对象。
4. 执行Mapper接口的方法，操作数据库，完成数据请求

```java
public class UserTest {
	
	private SqlSessionFactory sqlSessionFactory;

	@Before
	public void setUp() throws IOException{
		InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
		sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
	}
	
	@Test
	public void testGetUser(){
		SqlSession session = sqlSessionFactory.openSession();
		UserMapper userMapper = session.getMapper(UserMapper.class);
		User user = userMapper.getUser(1L);
		log.info(user.toString());
		session.close();
	}
}
```



## 二、mybatis整合spring

mybatis将对象的生命周期的管理交由spring来处理。

增加spring的配置，mybatis的代理对象由MapperFactoryBean实现。

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
	
	<!-- 数据源 -->
	<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
		<property name="url" value="jdbc:mysql://192.168.1.160:3306/lichao"></property>
		<property name="username" value="wms_dev"></property>
		<property name="password" value="wmswmsdevdev"></property>
	</bean>
	
	<!-- mybatis的Session工厂 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="configLocation" value="mybatis-config.xml"></property>
		<property name="dataSource" ref="dataSource"></property>
	</bean>
	
	<!-- UserMapper代理 -->
	<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
		<property name="mapperInterface" value="com.lcifn.orm.mybatis.mapper.UserMapper"/>
		<property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
	</bean>
</beans>
```

从spring上下文获取UserMapper对象，执行数据请求

```java
@Test
public void testUserMapper() {
    ApplicationContext context = new ClassPathXmlApplicationContext("spring-dao.xml");
    UserMapper userMapper = context.getBean(UserMapper.class);
    User user = userMapper.getUser(1L);
    log.info(user.toString());
}
```

下一章介绍mybatis配置文件的解析。