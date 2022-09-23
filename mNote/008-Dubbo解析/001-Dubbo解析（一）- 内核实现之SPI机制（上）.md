# Dubbo解析（一）- 内核实现之SPI机制（上）

> Dubbo采用微内核+插件的方式，使得设计优雅。扩展性强。但也给源码的学习带来一定的困难，初看者常常迷失在找不大方法的具体实现。在学习Dubbo源码前，博需要了解气内个的SPI机制。什么是SPI？他是JDK内部的一种服务发现机制，全称是Service Provider Interface（服务提供者接口），也是寻找接口服务的提供者的一种规范。

:cactus: 举个例子，你在淘宝上买东西，客户下单后要发快递，一般你会找快递公司帮你运输。快递公司有很多家，不同的快递公司提供的服务也不一样，比如顺丰可能就快点，韵达可能就慢点。发快读就是一个接口，而真正实现有不同的快读公司来完。

### 1. 定义一个发快递的接口

```java
package com.fish.dubbo.demo.service;

/**
 * 发快递接口
 *
 * @author wenbo
 * @date 2021/10/07 22:16
 **/
public interface Delivery {
    void deliveryProduct(String productName);
}
```

### 2. 有两个不同的实现

```java
package com.fish.dubbo.demo.service.impl;

import com.fish.dubbo.demo.service.Delivery;

/**
 * 顺丰快递
 *
 * @author wenbo
 * @date 2021/10/07 22:19
 **/
public class ShunFengDelivery implements Delivery {
    @Override
    public void deliveryProduct(String productName) {
        System.out.println("你的" + productName + "快递到家了，感谢您陪伴顺丰快递~");
    }
}
```

```java
package com.fish.dubbo.demo.service.impl;

import com.fish.dubbo.demo.service.Delivery;

/**
 * 韵达快递
 *
 * @author wenbo
 * @date 2021/10/07 22:20
 **/
public class YunDaDelivery implements Delivery {
    @Override
    public void deliveryProduct(String productName) {
        System.out.println("你的" + productName + "快递到家了，感谢您使用韵达快递~");
    }
}
```

因为快递公司是第三方，我们定义快递接口服务时，根本不知道发快递的到底是哪家公司。JDK的SPI机制就定义了一些规范，来帮助我们寻找服务的提供者实现：

- 在classpath的META-INF/services/目录下创建名称为服务接口的全限定名(package+className)的文件，文件编码必须为UTF-8，文件内容为服务接口实现类的全限定名，一个文件内可以定义多个实现类，按行分开即可。
- 服务接口实现类必须有一个无参的构造方法。
- 使用java.util包下的ServiceLoader类的load方法动态加载接口的实现类。

### 3. 按照JDK的SPI机制完成发快递的代码

```java
	package com.fish.dubbo.test;

import com.fish.dubbo.demo.service.Delivery;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Iterator;
import java.util.ServiceLoader;
import java.util.logging.LogManager;

/**
 * 快递测试类
 *
 * @author wenbo
 * @date 2021/10/07 22:32
 **/
public class DeliveryTest {

    @Test
    public void testDeliveryProduct(){
        String productName = "HUAWEI Mate 40 Pro";
        delivery(productName);
    }

    private void delivery(String productName) {
        ServiceLoader<Delivery> loader = ServiceLoader.load(Delivery.class);
        Iterator<Delivery> iterator = loader.iterator();
        while(iterator.hasNext()){
            Delivery delivery = iterator.next();
            delivery.deliveryProduct(productName);
        }
    }
}

// 输入结果如下：
你的HUAWEI Mate 40 Pro快递到家了，感谢您陪伴顺丰快递~
你的HUAWEI Mate 40 Pro快递到家了，感谢您使用韵达快递~
```

ServiceLoader实现了Iterable接口，可以遍历每一个服务提供者，并调用它的服务。结果就是客户买了一个HUAWEI Mate 40 Pro, 结果你快递了两个给人。当然淘宝卖家不会这么傻，一般都会默认一家快递公司，或者客户可以指定其他的快递公司，我们就需要接口中定义一个匹配方法。

```java
public interface Delivery {

    // 公司名称是否匹配
	boolean match(String companyName);
	void deliveryProduct(String productName);
}
```

那顺丰和韵达在服务实现类中就要匹配自己的公司名称

```java
// 顺丰
public class ShunFengDelivery implements Delivery{

	public boolean match(String companyName) {
		return "sf".equals(companyName);
	}
	
	public void deliveryProduct(String productName) {
		System.out.println("你的" + productName + "到家了，顺丰快递为您服务");
	}

}
```

```java
// 韵达
public class YunDaDelivery implements Delivery{

	public boolean match(String companyName) {
		return "yd".equals(companyName);
	}
	
	public void deliveryProduct(String productName) {
		System.out.println("你的" + productName + "到家了，韵达快递为您服务");
	}

}
```

发快递的代码也需要变更

```java
public class DeliveryTest {
	
	@Test
	public void testDeliveryProduct(){
		String productName = "Iphone X";
		String companyName = "sf";
		delivery(companyName, productName);
	}

	private void delivery(String companyName, String productName) {
		ServiceLoader<Delivery> loader = ServiceLoader.load(Delivery.class);
		Iterator<Delivery> iterator = loader.iterator();
		while(iterator.hasNext()){
			Delivery delivery = iterator.next();
			// 匹配指定的快递公司
			if(delivery.match(companyName)){
				delivery.deliveryProduct(productName);
				break;
			}
		}
	}
}

// 输出结果：
你的Iphone X到家了，顺丰快递为您服务
```

以上介绍了JDK的SPI机制，而Dubbo内核对于扩展点的加载方式就是从JDK的SPI机制增强而来的。那么JDK的SPI机制有什么缺点呢，Dubbo的SPI机制又增加了哪些呢，下一章见。