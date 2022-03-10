## Java 中的各种锁机制小叙

> ​       Java提供了种类丰富的锁，每种锁因其特性的不同，在适当的场景下，展现出非常高的效率。本着了解这些锁的实现原理，本系列文章旨在对锁相关的源码（本系列文章源码来自于JDK8）、使用场景进行举例，为自己梳理锁的知识点，以及不同锁的适用场景。

Java中往往是按照是否含有某一特性来定义锁，我们通过特性将锁分门别类，在使用对比的方式进行介绍，这样更容易理解相关知识。下边给出本系列文章总体的目录：
[Java锁机制大纲](https://img-blog.csdnimg.cn/20181122101753671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2F4aWFvYm9nZQ==,size_16,color_FFFFFF,t_70)
![本系列文章纲要](F:\PersonalWork\mwiki\mNote\library\003-Java篇\001-基础知识巩固\002-Java中锁机制小叙\099-images\001-outline.png)

