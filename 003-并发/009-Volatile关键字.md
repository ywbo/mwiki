## *`volatile`* 关键字

### 1. 如何保证变量的可见性？

> 在 Java 中，`Volatile` 关键字可以保证变量的可见性，如果我们将 变量声明成 *`volatile`*，这就指示 JVM，这个变量是共享且不稳定的，每次使用它都要到主存中进行读取。



![1664506668911](D:\OpenCode\mwiki-main\002-并发\1664506668911.png)

【发散】*`volatile`* 关键字其实并不是 Java 语言特有的，在 C 语言里也有，它最严实的意义就是禁用 CPU 的缓存。如果我们将一个变量使用 *`volatile`* 修饰，这就指示编译器，这个变量是共享且不稳定的，每次使用他都要主存中进行读取。

> *`volatile`* 关键字**能**保证数据的**可见性**，但**不能**保证数据的**原子性**。*`synchronized`* 关键字两者都能保证。

### 2. volatile 可以保证原子性吗？

> *`volatile`* 关键字**能保证变量的可见性，但不能保证变量的操作是原子性的。**

我们通过以下代码来进行说明？

```java
public class VolatileAtomicityDemo {
    public volatile static int inc = 0;

    public void increase(){
        inc++;
    }

    public static void main(String[] args) throws InterruptedException {
        ExecutorService threadPool = Executors.newFixedThreadPool(5);
        VolatileAtomicityDemo demo = new VolatileAtomicityDemo();
        for (int i = 0; i < 5; i++) {
            threadPool.execute(() -> {
                for (int j = 0; j < 500; j++) {
                    demo.increase();
                }
            });
        }

        // 等待1秒，保证上面的程序执行完成
        Thread.sleep(1000);
        System.out.println(inc);
        threadPool.shutdown();
    }
}
```

正常情况下，运行上面的代码理应输出 `2500`。但你真正运行了上面的代码之后，你会发现每次输出结果都小于 `2500`。

为什么会出现这种情况呢？不是说好了，`volatile` 可以保证变量的可见性嘛！

也就是说，如果 `volatile` 能保证 `inc++` 操作的原子性的话。每个线程中对 `inc` 变量自增完之后，其他线程可以立即看到修改后的值。5 个线程分别进行了 500 次操作，那么最终 inc 的值应该是 5*500=2500。

很多人会误认为自增操作 `inc++` 是原子性的，实际上，`inc++` 其实是一个复合操作，包括三步：

1. 读取 inc 的值。
2. 对 inc 加 1。
3. 将 inc 的值写回内存。

`volatile` 是无法保证这三个操作是具有原子性的，有可能导致下面这种情况出现：

1. 线程 1 对 `inc` 进行读取操作之后，还未对其进行修改。线程 2 又读取了 `inc`的值并对其进行修改（+1），再将`inc` 的值写回内存。
2. 线程 2 操作完毕后，线程 1 对 `inc`的值进行修改（+1），再将`inc` 的值写回内存。

这也就导致两个线程分别对 `inc` 进行了一次自增操作后，`inc` 实际上只增加了 1。

其实，如果想要保证上面的代码运行正确也非常简单，利用 `synchronized` 、`Lock`或者`AtomicInteger`都可以。

接下来我们使用 *`synchronized`* 关键字改进：

```java
public synchronized void increase(){
        inc++;
    }
```

使用 *`AtomicInteger`* 改进：

```java
public AtomicInteger inc = new AtomicInteger();

public void increase() {
    inc.getAndIncrement();
}
```

使用 *`ReentrantLock`* 改进：

```java
Lock lock = new ReentrantLock();
public void increase() {
    lock.lock();
    try {
        inc++;
    } finally {
        lock.unlock();
    }
}
```





