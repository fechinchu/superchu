# 并发编程03-Java并发容器和框架

# 1.ConcurrentHashMap

## 1.1.为什么使用ConcurrentHashMap

在并发编程中使用HashMap可能导致程序死循环,而使用线程安全的HashTable效率又非常低.HashMap在并发执行put操作时会引起死循环,是因为多线程会导致HashMap的Entry链表形成环形数据结构,一旦形成环形数据结构,Entry的Next节点永远不为空,就会产生死循环获取Entry.

ConcurrentHashMap使用锁分段技术.首先将数据分成一段一段地存储,然后给每一段数据搭配一把锁,当一个线程占用锁访问其中一个段数据的时候,其他段的数据也能被其他线程访问.

## 1.2.ConcurrentHashMap结构

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成.Segment是一种可重入锁(ReentrantLock),在ConcurrentHashMap中扮演锁的角色.HashEntry则用于存储键值对数据.一个ConcurrentHashMap里包含一个Segment数组.Segment的结构和HashMap类似.是一种数组和链表结构.一个Segment里包含一个HashEntry数组.每个HashEntry是一个链表结构的数据,每个Segment守护着一个HashEntry数组里的元素,当对HashEntry是一个链表结构的元素进行修改时,必须首先获得与它对应的Segment锁;

# 2.ConcurrentLinkedQueue

实现一个线程安全的队列有两种方式:

* 阻塞算法:可以使用一个锁(入队和出队用同一把锁)或两个锁(入队和出队用不同的锁)等方式来实现;
* 非阻塞算法:可以使用循环CAS的方式来实现.

ConcurrentLinkedQueue是一个基于连接节点的无界线程安全队列.它采用先进先出的规则对节点进行排序,当我们添加一个元素的时候,它会添加到队列的尾部;当我们获取一个元素时候,它会返回队列的头部元素,使用CAS算法来实现.

## 2.1.ConcurrentLinkedQueue结构

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210514162741796.png" alt="image-20210514162741796" style="zoom:67%;" />



ConcurrentLinkedQueue由head节点和tail节点组成,每个节点Node由节点元素(item)和指向下一个节点(next)的引用组成,节点与节点之间就是通过这个next关联起来,从而组成一张链表结构的队列.

默认情况下head节点存储的元素为空.tail节点等于head节点.

## 2.2.入队列

![image-20210514163945394](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210514163945394.png)

如上图:

1. 添加元素1.队列更新head节点的next节点为元素1节点.
2. 添加元素2.队列首先设置元素1节点的next节点为元素2节点,然后更新tail节点指向元素2节点;
3. 添加元素3.设置tail节点的next节点为元素3节点;
4. 添加元素4.设置元素3的next节点为元素4节点,然后将tail节点指向元素4节点;

通过调试入队过程并观察head节点和tail节点的变化,发现入队主要做两件事情:

1. 将入队节点设置成当前队列尾节点的下一个节点;
2. 更新tail节点,如果tail节点的next节点不为空,则将入队节点设置成tail节点.如果tail节点的next节点为空,则将入队节点设置成tail的next节点;

![image-20210514165210837](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210514165210837.png)

