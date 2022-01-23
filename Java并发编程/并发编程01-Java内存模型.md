#并发编程01-并发理论基础

# 1.可见性,原子性和有序性问题

## 1.1.并发程序的幕后

在CPU,内存,I/O设备不断迭代的时候,有一个核心矛盾一直存在,就是这三者的速度差异.为了合理利用CPU的高性能,平衡这三者的速度差异:

1. CPU增加了缓存,以均衡与内存的速度差异;
2. 操作系统增加了进程,线程,以分时复用CPU,进而均衡CPU与I/O设备的速度差异;
3. 编译程序优化指令执行次序,使得缓存能够得到更加合理的利用;

## 1.2.缓存导致的可见性问题

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506112326983.png" alt="image-20210506112326983" style="zoom: 50%;" />

多核时代,每颗CPU都有自己的缓存,当多个线程在不同的CPU上执行时候,这些线程操作的是不同的CPU缓存,比如上图,线程A操作的是CPU-1上的缓存,而线程B操作的是CPU-2上的缓存,这个时候线程A对变量A的操作对于线程B而言不具备可见性;

## 1.3.线程切换导致的原子性问题

现代的操作系统都是基于线程来进行调度,而一个进程创建的所有线程,都是共享一个内存空间.操作系统做任务切换,可以发生在任何一条CPU指令执行完.我们把一个或者多个操作在CPU执行的过程中不被中断的特行称为原子性.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506133644361.png" alt="image-20210506133644361" style="zoom:50%;" />

## 1.4.编译优化导致的有序性问题

有序性指的是程序按照代码的先后顺序执行,编译器为了优化性能,有时候会改变程序中语句的先后顺序.

~~~java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}
~~~

上述例子看似没有问题.但问题出在new操作上.我们以为的new操作时:

1. 分配一块内存M;
2. 在内存M上初始化Singleton对象;
3. 然后M的地址赋值给instance变量;

然而实际上优化后的执行路径是:

1. 分配一块内存M;
2. 将M的地址赋值给instance变量;
3. 最后在内存M上初始化Singleton对象;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506134928545.png" alt="image-20210506134928545" style="zoom:50%;" />

# 2.Java内存模型

## 2.1.Java内存模型的抽象结构

在Java中,所有实例,静态域和数组元素都存储在堆内存中,堆内存在线程之间共享,简称共享变量.局部变量,方法定义参数和异常处理参数不会在线程之间共享,它们不会有内存可见性问题,也不受内存模型的影响;

Java线程之间的通信由Java内存模型控制,JMM决定了一个线程对共享变量的写入何时对另一个线程可见.从抽象的角度来看,JMM定义了线程和主内存之间的抽象关系:**线程之间的共享变量存储在主内存中,每个线程都有一个私有的本地内存,本地内存中存储了该线程以读/写共享变量的副本.本地内存是JMM的一个抽象概念,并不真实存在,它覆盖了缓存,写缓冲区,寄存器以及其他硬件和编译器优化.**

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506140852169.png" alt="image-20210506140852169" style="zoom:50%;" />

## 2.2.重排序

重排序是指编译器和处理器为了优化程序性能而堆指令序列进行重新排序的一种手段.

如果两个操作访问同一个变量,且这两个操作中有一个为写操作,此时这两个操作之间就存在数据依赖性.编译器和处理器可能会对操作做重排序.编译器和处理器在重排序时,会遵守数据依赖性,编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序.这里说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作.不同处理器之间的不同线程之间的数据依赖性不被编译器和处理器考虑.

**as-if-serial语义**:**不管怎么重排序,单线程程序的执行结果不能被改变**

为了提高性能,编译器和处理器会对指令做重排序,分3种类型:

> 1. 编译器优化的重排序.编译器在不改变单线程程序语义的前提下,可以重新安排语句的执行顺序;
> 2. 指令级并行的重排序,现代处理器采用了指令级并行技术将多条指令重叠执行.如果不存在数据依赖性,处理器可以改变语句对应机器指令的执行顺序;
> 3. 内存系统的重排序.由于处理器使用缓存和读/写缓冲区,这使得加载和存储操作看上去是乱序执行.

上述的1属于编译器重排序,2和3属于处理器重排序

1. 对于编译器,JMM的编译器重排序规则会禁止特定类型的编译器重排序(不是所有的编译器重排序都要禁止);
2. 对于处理器,JMM的处理器重排序规则会要求Java编译器在生成指令序列时,插入特定类型的内存屏障指令,通过内存屏障指令来禁止特定类型的处理器重排序.

## 2.3.并发编程模型的分类

现代的处理器使用写缓冲区临时保存向内存写入的数据.通过以批处理的方式属性写缓冲区,以及合并写缓冲区对同一内存地址的多次写,减少对内存总线的占用.虽然写缓冲区有那么多好处,但**每个处理器上的写缓冲区,仅仅对它所在的处理器可见**.这个特性会对内存操作的执行顺序产生重要影响:**处理器对内存的读/写操作的执行顺序,不一定与内存实际发生的读/写顺序一致**.

为了保证内存可见性,Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的重排序.

![image-20210506144547731](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506144547731.png)

StoreLoad Barriers是一个全能型的屏障,它同时具有其他3个屏障的效果.现代的处理器大多支持该屏障.

## 2.4.happens-before简介

从JDK5开始,Java使用新的JSR-133内存模型,JSR-133使用happens-before概念来阐述操作之间的内存可见性,在JMM中,如果一个操作执行的记过需要对另一个操作可见,那么这两个操作之间必须要存在happens-before关系.这里的两个操作可以是在一个线程内,也可以是在不同线程之间.规则如下:

> * 程序顺序规则:一个线程中的每个操作,happens-before于该线程中的任意后续操作;
> * 监视器锁规则:对一个锁的解锁,happens-before于随后对这个锁的加锁;
> * volatile变量规则:对一个volatile域的写,happens-before与任意后续对这个volatile域的读;
> * 传递性:如果A happens-before B,且B hanppens-before C,那么A happens-before C;
> * `start()`规则:如果线程A执行操作`ThreadB.start()`,那么A线程的`ThreadB.start()`操作happens-before于线程B中的任意操作;
> * `join()`规则:如果线程A执行操作`ThreadB.join()`,那么B线程中任意的操作happens-before于线程A从`ThreadB.join()`操作成功返回;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506145504809.png" alt="image-20210506145504809" style="zoom:50%;" />

JSR133对happens-before关系的定义如下:

1. 如果一个操作hanppens-before另一个操作,那么第一个操作的执行结果将对对二个操作可见,而且第一个操作的执行顺序排在第二个操作之前;
2. 两个操作之间存在happens-before关系,并不意味着java平台的具体实现必须要按照happens-before关系指定的顺序来执行.如果重排序之后的执行结果,与按happens-before关系来执行的结果一直,那么这种重排序并不非法;

因此,happens-before关系本质上和as-if-serial是一回事;

> as-if-serial语义保证单线程内程序的执行结果不被改变,happens-before关系保证的正确去同步的多线程程序的执行结果不被改变;

# 3.volatile内存语义

## 3.1.volatile的特性

* 可见性:对于一个volatile变量的读,总是能看到人以线程对这个volatile变量最后的写入;
* 原子性:对任意单个volatile变量的读写具有原子性,但类似与volatile++这种复合操作不再具有原子性;

## 3.2.volatile写-读建立的happens-before关系

~~~java
public class VolatileDemo {
     int a = 0;
     volatile boolean flag = false;

     public void writer(){
         a = 1;
         flag = true;
     }

     public void reader(){
         if(flag){
             int i = a;
         }
     }
}
~~~

假设线程A执行`writer()`方法之后,线程B执行`reader()`方法,根据happens-before规则如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506162746922.png" alt="image-20210506162746922" style="zoom:50%;" />

## 3.3.volatile写-读的内存语义

* volatile写的内存语义:**当写一个volatile变量时,JMM会把该线程对应的本地内存中的共享变量值刷新到主内存**;

* volatile读的内存语义:**当读一个volatile变量时,JMM会把该线程对应的本地内存置为无效,线程接下来将从主内存中读取共享变量**;
* 综合来看:在读线程B读一个volatile变量后,写线程A在写这个volatile变量之前所有可见的共享变量的值都将立即变得对读线程B可见;

## 3.4.volatile内存语义的实现

### 3.4.1.编译器实现

为了实现volatile内存语义,JMM会分别限制这两种类型的重排序类型,下图是规则表:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506163729436.png" alt="image-20210506163729436" style="zoom:50%;" />

* 当第二个操作是volatile写时,不管第一个操作是什么,都不能重排序.这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后;
* 当第一个操作是volatile读时,不管第二个操作是什么,都不能重排序,这个规则确保volatile读之后的操作不会被编辑器重排序到volatile读之前;
* 当第一个操作是volatile写,第二个操作时volatile读,不能重排序;

### 3.4.2.处理器实现

如下是保守策略的JMM内存屏障插入策略:

* 在每个volatile写操作的前面插入一个StoreStore屏障;
* 在每个volatile写操作的后面插入一个StoreLoad屏障;
* 在每个volatile读操作的后面插入一个LoadLoad屏障;
* 在每个volatile读操作的后面插入一个LoadStore屏障;

# 4.锁的内存语义

## 4.1.锁的释放-获取建立的happens-before关系

~~~java
public class MonitorExample {
    
    int a = 0;
    
    public synchronized void writer(){
        a++;
    }
    
    public synchronized void reader(){
        int i = a;
    }
}
~~~

假设线程A执行`writer()`方法,随后线程B执行`reader()`方法

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506173715198.png" alt="image-20210506173715198" style="zoom: 67%;" />

## 4.2.锁的释放-获取的内存含义

* 当线程释放锁时,JMM会把该线程对应的本地内存中的共享变量刷新到主内存中;
* 当线程获取锁时,JMM会把该线程对应的本地内存置为无效,从而使得被监视器保护的临界区代码必须从主内存中读取共享变量;

## 4.3.锁内存语义的实现

### 4.3.1.公平锁

在`ReentrantLock`中,调用`lock()`方法获取锁,调用`unlock()`方法释放锁.`ReentrantLock`的实现依赖于Java同步器框架`AbstractQueuedSynchronizer`(AQS).AQS使用一个整型的volatile变量(state)来维护同步状态;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506174911152.png" alt="image-20210506174911152" style="zoom: 67%;" />

ReentrantLock分为公平锁和非公平锁,使用公平锁时候,加锁方法`lock`调用轨迹如下:

1. `ReentrantLock`:`lock()`;
2. `FairSync`:`lock()`;
3. `AbstractQueuedSynchronizer`:`acquire(int arg)`;
4. `ReentrantLock`:`tryAcquire(int acquires)`;

![image-20210506175505095](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506175505095.png)

释放锁方法`unlock()`调用轨迹如下:

1. `ReentrantLock:unlock()`;
2. `AbstractQueuedSynchronizer:release(int arg)`;
3. `Sync:tryRelease(int releases)`;

![image-20210506175747891](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506175747891.png)

公平锁在释放锁的最后写volatile变量state,在获取锁时首先读这个volatile变量.根据volatile的happens-before规则,释放锁的线程在写volatile变量之前可见的共享变量.在获取获取锁的线程读取同一个volatile变量将立即变得对获取锁的线程可见.

### 4.3.2.非公平锁

非公平锁的释放和公平锁一样,非公平锁的加锁`lock()`调用轨迹如下:

1. `ReentrantLock:lock()`;
2. `NonfairSync:lock()`;
3. `AbstracrtQueuedSynchronizer:compareAndSetState(int expect,int update)`;

![image-20210506183408917](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506183408917.png)

### 4.3.3.总结

* 公平锁和非公平锁释放时,最后都要写一个volatile变量state;
* 公平锁获取时,首先会读volatile变量;
* 非公平锁获取时,首先会用CAS更新volatile变量,这个操作同时具有volatile读和volatile写的内存含义;

## 4.4.concurrent包的实现

由于Java的CAS同时具有volatile读和volatile写的内存语义,因此Java线程之间的通信由下面的四种方式:

1. A线程写volatile变量,随后B线程读这个volatile变量;
2. A线程写volatile变量,随后B线程用CAS更新这个volatile变量;
3. A线程用CAS更新一个volatile变量,随后B线程用CAS更新这个volatile变量;
4. A线程用CAS更新一个volatile变量,随后B线程读这个volatile变量;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506185626742.png" alt="image-20210506185626742" style="zoom:50%;" />

分析concurrent包的源代码实现,会发现一个通用化的实现模式:如上图:

1. 首先声明共享变量为volatile;
2. 然后使用CAS的原子条件更新来实现线程之间的同步,同时配合以volatile的读/写和CAS说具有的volatile读和写内存语义来实现线程间的通信;

# 5.final域的内存语义

## 5.1.final域的重排序规则

1. 在构造函数内对一个final域的写入,与随后把这个被构造的对象的引用赋值给一个引用变量,两个操作之间不能重排序;
2. 初次读一个包含final域的对象的引用,与随后初次读这个final域,这两个操作之间不能重排序;

~~~java
public class FinalExample {

    int i;

    final int j;

    static FinalExample obj;

    public FinalExample(){
        i = 1;
        j = 2;
    }

    public static void write(){
        obj = new FinalExample();
    }

    public static void read(){
        FinalExample object = obj;
        int a = object.i;
        int b = object.j;
    }
}
~~~

假设一个线程A执行`write()`方法,随后另一个线程B执行`read()`方法.

## 5.2.写final域的重排序规则

写final域的重排序规则禁止把final域的写重排序到构造函数之外.这个规则的实现包含下面两个方面:

1. JMM禁止编译器把final域的写重排序到构造函数之外;
2. 编译器会在final域的写之后,构造函数return之前,插入一个StoreStore屏障.这个屏障禁止处理器把final域的写重排序到构造函数之外;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506232418290.png" style="zoom: 33%;" />

在上图中,写普通域的操作被编译器重排序到了构造函数之外,读线程B错误的读取了普通变量i初始化之前的值,而协final域的重排序规则限定在了构造函数之内,读线程B正确的读取了final变量初始化之后的值;

写final域的重排序规则可以确保,在对象引用为任意线程可见之前,对象final域已经被正确初始化过了;

## 5.3.读final域的重排序规则

1. 对于处理器来说,初次读对象引用与初次读该对象包含的final域,JMM禁止处理器重排序这两个操作;
2. 编译器在读final域操作的前面插入一个LoadLoad屏障;

初次读对象引用与初次读该对象包含的final域,这两个操作之间存在简介依赖关系,因此编译器不会重排序这两个操作,但有少数处理器允许对存在间接依赖关系的操作做重排序,这个规则专门用来针对这种处理器;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210506233835363.png" alt="image-20210506233835363" style="zoom: 33%;" />

## 5.4.final域为引用类型

新增约束:在构造函数内对一个final引用的对象的成员域的写入,与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量,这两个操作之间不能重排序;



