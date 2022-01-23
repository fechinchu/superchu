# 并发编程04-Java并发工具

# 1.Java中的原子操作类

## 1.1.原子更新基本类型类

使用原子的方式更新基本类型,Atomic包提供了如下:

* AtomicBoolean:原子更新布尔类型;
* AtomicInteger:原子更新整型;
* AtomicLong:原子更新长整型;

Atomic包提供了3种基本类型的原子更新,但是Java的基本类型还有`char`,`float`和`double`等.如何原子更新其他基本类型呢?Atomic包里的类基本都是使用`Unsafe`实现的

![image-20210522212210887](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210522212210887.png)

`Unsafe`只提供了3种CAS方法如上,观察`AtomicBoolean`代码如下:它是将Boolean转成整形.再使用`compareAndSwapInt`进行CAS.所以原子更新char,float和double变量也可以用类似的思路进行实现.

![image-20210522212406005](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210522212406005.png)

## 1.2.原子更新数组

* AtomicIntegerArray:原子更新整型数组里的元素;
* AtomicLongArray:原子更新长整型数组里的元素;
* AtomicReferenceArray:原子更新引用类型数组里的元素;

```java
public class AtomicArrayTest {

    static int[] value = new int[]{1,2};

    static AtomicIntegerArray ai = new AtomicIntegerArray(value);

    public static void main(String[] args) {
        ai.getAndSet(0,3);
        System.out.println(ai.get(0));//结果:3
        System.out.println(value[0]);//结果:1
    }
}
```

需要注意的是:数组value通过构造方法传递进去,然后AtomicIntegerArray会将当前数组复制一份,所以当AtomicIntegerArray对内部的数组元素进行修改时,不会影响传入的数组.

## 1.3.原子更新引用类型

* AtomicReference:原子更新引用类型;
* AtomicReferenceFieldUpdater:原子更新引用类型里的字段;
* AtomicMarkableReference:原子更新带有标记位的引用类型.可以原子更新一个布尔类型的标记为和引用类型

~~~java
package org.fechin.jmm;

import java.util.concurrent.atomic.AtomicMarkableReference;

/**
 * @Author:guoqing.zhu
 * @Date：2021/5/22 22:08
 * @Desription: TODO
 * @Version: 1.0
 */
public class AtomicMarkableReferenceTest {
    
    public static void main(String[] args) {
        User user3 = new User("张三",3);
        User user4 = new User("李四", 4);
        AtomicMarkableReference<User> atomicUserRef = new AtomicMarkableReference<>(user3,false);
        boolean b = atomicUserRef.compareAndSet(user3, user4, false, true);
        System.out.println(b);
        System.out.println(atomicUserRef.getReference());
    }

    static class User{
        private String name;
        private Integer age;

        public User(String name, Integer age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public Integer getAge() {
            return age;
        }

        public void setAge(Integer age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "User{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}
~~~

## 1.4.原子更新字段类

* AtomicIntegerFieldUpdater:原子更新整型的字段的更新器;
* AtomicLongFieldUpdater:原子更新长整型字段的更新器;
* AtomicStampedReference:原子更新带有版本号的引用类型,该类型将整数值与引用观念起来,可用于原子的更新数据和数据版本号,可以解决CAS进行原子更新出现的ABA问题;

~~~java
package org.fechin.jmm;

import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * @Author:guoqing.zhu
 * @Date：2021/5/22 22:20
 * @Desription: TODO
 * @Version: 1.0
 */
public class AtomicStampedReferenceTest {

    public static void main(String[] args) {
        User user3 = new User("张三",3);
        User user4 = new User("李四", 4);
        AtomicStampedReference<User> a = new AtomicStampedReference<User>(user3,1);
        boolean b = a.compareAndSet(user3, user4, 1, 2);
        System.out.println(b);
        System.out.println(a.getReference());
    }

    static class User{
        private String name;
        private Integer age;

        public User(String name, Integer age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public Integer getAge() {
            return age;
        }

        public void setAge(Integer age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "User{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}
~~~

# 2.Java中的并发工具类

## 2.1.等待多线程完成的CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成操作.

~~~java
public class CountDownLatchTest {
    static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(1);
                countDownLatch.countDown();
                System.out.println(2);
                countDownLatch.countDown();
            }
        }).start();

        countDownLatch.await();
        System.out.println(3);

    }
}
~~~

当我们调用CountDownLatch的countDown方法时,N就会减1,CountDownLatch的await方法会阻塞当前线程,直到N变成0,由于countDown放阿飞可以用在任何地方,所以这里的N可以是N个线程,也可以是1个线程里的N个执行步骤.如果我们不想让主线程一直等待,所以可以使用另外一个带指定时间的`await(long time,TimeUnit unit)`;这个方法等待待定时间后,就会不再阻塞当前线程.

注意:计数器必须大于等于0,只是等于0时候,计数器就是0,调用await方法时不会阻塞当前线程.

## 2.2.同步屏障CyclicBarrier

同步屏障做的事情是,让一组线程到达一个屏障(也叫同步点)时被阻塞,直到最后一个线程到达屏障时,屏障才会开门,所有被屏障拦截的线程才会执行.

~~~java
public class CyclicBarrierTest {

    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(1);
            }
        }).start();

        c.await();
        System.out.println(2);
    }
}
~~~

`CyclicBarrier(int parties,Runnable barrierAction)`,用于在线程到达屏障时,优先执行barrierAction,方便处理更复杂的业务场景.

## 2.3.控制并发线程数Semaphore

Semaphore是用来控制同时访问特定资源的线程数量,它通过协调各个线程,以保证合理的使用公共资源.

Semaphore可以用于做流量控制,特别是公共资源有限的应用场景,比如数据库连接.

首先Semaphore首先线程使用Semaphore的`acquire()`方法获取一个许可证,使用完后调用`release()`方法归还许可证.

~~~java
package org.fechin.jmm;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

/**
 * @Author:guoqing.zhu
 * @Date：2021/5/22 23:13
 * @Desription: TODO
 * @Version: 1.0
 */
public class SemaphoreTest {

    private static final int THREAD_COUNT = 30;

    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            int finalI = i;
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("do something"+ finalI);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    s.release();
                }
            });
        }

        threadPool.shutdown();
    }
}
~~~

## 2.4.线程间交换数据Exchanger

Exchanger是一个用于线程间协作的工具类,Exchanger用于进行进程间的数据交换.它提供一个同步点,在这个同步点,两个线程可以交换彼此的数据.这两个线程通过exchange方法交换数据.如果第一个线程先执行`exchange()`方法,它会一直等待第二个线程也执行`exchange()`方法,当两个线程都到达同步点时候,这两个线程就可以交换数据.

~~~java
package org.fechin.jmm;

import java.util.concurrent.Exchanger;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @Author:guoqing.zhu
 * @Date：2021/5/22 23:34
 * @Desription: TODO
 * @Version: 1.0
 */
public class ExchangerTest {

    private static final Exchanger<String> EXCHANGER = new Exchanger<>();

    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                String A = "A do";
                try {
                    String exchange = EXCHANGER.exchange(A);
                    System.out.println(exchange+A);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                String B = "B do";
                try {
                    String exchange = EXCHANGER.exchange(B);
                    System.out.println(exchange+B);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        threadPool.shutdown();
    }
}
~~~

如果两个线程有一个没有执行`exchange()`方法,则会一直等待,如果担心有特殊情况放生,避免一直等待,可以使用`exchange(V x,long timeout,TimeUnit unit)`;

# 3.Java中的线程池

## 3.1.线程池实现原理

* 线程池中刚开始没有线程,当一个任务提交给线程池后,线程池会创建一个新的线程来执行任务;
* 当线程数达到corePoolSize并没有线程空闲时,这时候再加入任务,新加的任务会被加入workQueue队列排队,直到有空闲的线程;
* 如果队列选择了有界队列,那么任务超过了队列大小时,会创建maximumPoolSize-corePoolSize数目的线程来救急;
* 如果线程到达maximumPoolSize仍然有新任务这时候会执行拒绝策略.拒绝策略JDK提供了4种实现,其他框架也提供了实现:
  * AbortPolicy让调用这抛出RejectedExecutionException异常,这是默认策略;
  * CallerRunsPolicy让调用者运行任务;
  * DiscardPolicy放弃本次任务;
  * DiscardOldestPolicy:放弃队列中最早的任务,本任务取代它;
  * Dubbo的实现,在抛出RejectedExecutionException异常之前会记录日志,并dump线程栈信息,方便定位问题;
  * Netty的实现,是创建一个新线程来执行任务;
  * ActiveMQ的实现,带超时等待(60s)尝试放入队列:
  * PinPoint的实现,它使用了一个拒绝策略链,会逐一尝试策略链中每种拒绝策略;
* 当高峰过去后,超过corePoolSize的救急线程如果一段时间没有任务做,需要结束节省资源,这个时间由keepAliveTime和unit来控制;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210522235103457.png" alt="image-20210522235103457" style="zoom:67%;" />

## 3.2.线程池的使用

![image-20210522235944799](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210522235944799.png)

* runnableTaskQueue(任务队列):用于保存等待执行的任务的阻塞队列.可以选择一下几个阻塞队列.
  * ArrayBlockingQueue:是一个基于数组结构的有界队列,此队列按FIFO(先进先出)原则对元素进行排序;
  * LinkedBlockingQueue:一个基于链表结构的阻塞队列,此队列按FIFO排序元素,吞吐量通常要高于ArrayBlockingQueue;
  * SynchronousQueue:一个不存储元素的阻塞队列.每个插入操作必须等到另一个线程调用移除操作,否则插入操作一直阻塞转状态;
  * PriorityBlockingQueue:具有优先级的无限阻塞队列;
* ThreadFactory:用于设置创建线程的工厂,可以通过线程工厂给每个创建出来的线程设置有意义的名字.使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池设置有意义的名字.`new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build()`;
* RejectedExecutionHandler:默认策略是AbortPolicy:
  * AbortPolicy:直接抛出异常;
  * CallerRunsPolicy:只用调用者所在线程来运行任务;
  * DiscardOldestPolicy:丢弃队列里最近的一个任务,并执行当前任务;
  * DiscardPolicy:不处理,丢弃掉;

## 3.3.向线程池提交任务

![image-20210523001913715](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523001913715.png)

## 3.4.线程池的监控

ThreadPoolExecutor属性如下:

* taskCount:线程池需要执行的任务数量;
* completedTaskCount:线程池在运行过程中已完成的任务数量,小于或等于taskCount;
* largestPoolSize:线程池里曾经创建过的最大线程数量;
* getPoolSize:线程池的线程数量.
* getActiveCount:获取活动的线程数量;

通过扩展线程池进行监控.可以通过继承线程池来自定义线程池.重写线程池的`beforeExecute()`,`afterExecute()`,`terminated()`方法,也可以在任务执行前,执行后和线程池关闭前执行一些代码来进行监控.

# 4.Executor框架

## 4.1.Executor框架简介

### 4.1.1.Executor框架的两级调度模型

在HotSpot VM的线程模型中,Java线程(java.lang.Thread)被一对一映射为本地操作系统线程.Java线程启动时会创建一个本地操作系统线程.当该Java线程终止时,这个操作系统线程也会被回收.操作系统会调度所有线程并将它们分配给可用的CPU;

在上层,Java多线程程序通常把应用分解为若干个任务,然后使用用户级的调度器Executor框架将这些任务映射为固定数量线程;在底层,操作系统将这些线程映射到硬件处理器上.

### 4.1.2.Executor框架的结构与成员

####4.1.2.1.Executor框架的结构

Executor框架主要有3大部分组成:

1. 任务:包括被执行任务需要实现的接口:`Runnable`或`Callable`;
2. 任务的执行:`Executor`,`ExecutorService`, `ThreadPoolExecutor`和`ScheduledThreadPoolExecutor`;
3. 异步计算的结果,包括`Future`和`FutureTask`;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523005716885.png" alt="image-20210523005716885" style="zoom:67%;" />

Executor框架的类与接口如上;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523005812315.png" alt="image-20210523005812315" style="zoom:67%;" />

Executor框架的使用如上;

#### 4.1.2.2.Executor框架的成员

1. `ThreadPoolExecutor`:ThreadPoolExecutor通常使用工厂类Executors来创建.Executors可以创建3种类型的ThreadPoolExecutors:`SingleThreadExecutor`,`FixedThreadPool`,`CachedThreadPool`;
   1. `FixedThreadPool`:创建使用固定线程数.适用于为了满足资源管理的需求,需要限制当前线程数量的应用场景,适用于负载比较重的服务器;
   2. `SingleThreadExecutor`:创建使用单个线程数量,适用于需要保证顺序执行各个任务;并且在任意时间点,不会有多个线程是活动的应用场景;
   3. `CachedThreadPool`:创建一个会根据需要创建新线程的,CachedThreadPool是大小无界的线程池,适用于执行很多的短期异步任务的小程序,或者是负载较轻的服务器;
2. `ScheduledThreadPoolExecutor`:可以创建2种类型的ScheduledThreadPoolExecutor如下:
   1. `ScheduledThreadPoolExecutor`:包含若干个线程的ScheduledThreadPoolExecutor;
   2. `SingleThreadScheduledExecutor`:只包含一个线程的ScheduledThreadPoolExecutor;

## 4.2.ThreadPoolExecutor

### 4.2.1.FixedThreadPool

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523192500016.png" alt="image-20210523192500016" style="zoom:50%;" />

FixedThreadPool被称为可重用固定线程数的线程池.FixedThreadPool的corePoolSize和maximumPoolSize都被设置为创建FixedThreadPool时指定的参数nThreads.当线程池中的线程数大于corePoolSize时,keepAliveTime为多余的空闲线程等待新任务的最长时间,这里的keepAliveTime为0,意味着多余的空闲线程会被立即终止.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523193927309.png" alt="image-20210523193927309" style="zoom: 50%;" />

1. 如果当前运行的线程数少于corePoolSize,则创建新线程来执行;
2. 在线程池完成预热之后(当前运行的线程数等于corePoolSize),将任务加入LinkedBlockingQueue;
3. 线程执行完1中的任务后,会在循环中反复从LinkedBlockingQueue获取任务来执行;FixedThreadPool使用无界队列LinkedBlockingQueue作为线程池的工作队列(队列的容量为Integer.MAX_VALUE),使用无界队列作为工作队列会对线程池带来如下影响;
   1. 当线程池中的线程数达到corePoolSize后,新任务将在无界队列中等待,因此线程池中的线程数不会超过corePoolSIze;
   2. 由于1,使用无界队列时maximumPoolSize将是一个无效参数;
   3. 由于1和2,使用无界队列时keepAliveTime将是一个无效参数;
   4. 由于使用无界队列,运行中的FixedThreadPool不会拒绝任务;

### 4.2.2.SingleThreadExecutor

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523195207249.png" alt="image-20210523195207249" style="zoom:67%;" />

SingleThreadExecutor的corePoolSize和maximumPoolSize被设置为1.其他参数与FixedThreadPool相同,SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523195451121.png" alt="image-20210523195451121" style="zoom:67%;" />

1. 如果当先运行的线程数小于corePoolSize(即线程池中无运行的线程),则创建一个新线程来执行任务;
2. 在线程池完成预热之后(当前线程池中有一个运行的线程),将任务加入LinkedBlockingQueue;
3. 线程执行完1中的任务后,会在一个无限循环中反复从LinkedBlockingQueue获取任务来执行;

### 4.2.3.CachedThreadPool

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523195717284.png" alt="image-20210523195717284" style="zoom:67%;" />

CachedThreadPool的corePoolSize被设置为0,即corePool为空;maximumPoolSize被设置为Integer.MAX_VALUE.即maximumPool是无界的,这里把keepAliveTime设置为60L,意味着CachedThreadPool中的空闲线程等待新任务的最长时间为60秒,空闲线程超过60秒将会被终止;

CachedThreadPool使用没有容量的SynchronousQueue作为线程池的工作队列,但CachedThreadPool的maximumPool是无界的.这意味着,如果主线程提交任务的速度高于maximumPool中线程处理任务的速度时,CachedThreadPool会不断创建新的线程.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210523202150442.png" alt="image-20210523202150442" style="zoom:67%;" />

1. 首先执行`SynchronousQueue.offer(Runnable task)`.如果当前maximumPool中有空闲线程正在执行`SynchronousQueue.poll(keepAliveTime,TimeUnit)`,那么主线程执行offer操作与空闲线程执行的poll操作配对成功,主线程把任务交给空闲线程执行,`execute()`方法执行完成,否则执行2;
2. 当初次华maximumPool为空,或者maximumPool中当前没有空闲线程时,将没有线程执行`SynchrousQueue.poll(keepAliveTime,TimeUnit)`.这种情况下,步骤1失败,此时CachedThreadPool会创建一个新线程执行任务,`execute()`方法执行完成;
3. 在步骤2中创建的线程将任务执行完成后,会执行`SynchronousQueue.poll(keepAliveTime,TimeUnit)`这个poll操作会让空闲线程最多在SynchronousQueue中等待60秒,然后终止;

## 4.3.ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor继承自ThreadPoolExecutor,主要用来在给定的延迟后的运行任务,或者定期执行的任务.

### 4.3.1.ScheduledThreadPoolExecutor的运行机制

1. ScheduledThreadPoolExecutor的`scheduleAtFixedRate()`方法或者`scheduleWithFixedDelay()`方法时,会向ScheduledThreadPoolExecutor的DelayQueue添加一个实现了RunnableScheduledFuture接口的ScheduledFutureTask;
2. 线程池中的线程从DelayQueue中获取ScheduledFutureTask,然后执行任务;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210524111733639.png" alt="image-20210524111733639" style="zoom:67%;" />

ScheduledThreadPoolExecutor为了实现周期性的执行任务,对ThreadPoolExecutor做了如下修改:

* 使用DelayQueue作为任务队列;
* 获取任务的方式不同;
* 执行周期任务后,增加了额外任务;

### 4.3.2.ScheduledThreadPoolExecutor的实现

ScheduledThreadPoolExecutor会把待调度的任务ScheduledFutureTask放到一个DelayQueue中.ScheduledFutureTask主要包含3个成员变量,如下:

* long型成员变量time,表示这个任务将要被执行的具体时间;
* long型成员变量sequenceNumber,表示这个任务被添加到ScheduledThreadPoolExecutor中的序号;
* long型成员变量period,表示任务执行的隔离周期;

DelayQueue封装一个PriorityQueue,这个PriorityQueue会对队列中的ScheduledFutureTask进行排序.排序时,time小的排在前面,如果两个ScheduledFutureTask的time相同,就比较sequenceNumber,sequenceNumber小的排在前面.

## 4.4.FutureTask

### 4.4.1.FutureTask简介

Future接口和实现Future接口的FutureTask类,代表异步计算的结果.

FutureTask除了实现Future接口外,还实现Runnable接口.因此,FutureTask可以交给Executor执行,也可以由调用线程直接执行(`FutureTask.run()`).根据`FutureTask.run()`方法被执行的时机,分为3种状态为未启动,已启动和已完成.

当FutureTask处于未启动或已启动状态,执行`FutureTask.get()`方法将导致调用线程阻塞,当FutureTask处于已完成状态时候,执行`FutureTask.get()`方法将导致调用线程立即返回结果或抛出异常.

### 4.4.2.FutureTask的实现

早期FutureTask的实现基于AQS.

![image-20210524140751513](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210524140751513.png)

![image-20210524141412644](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210524141412644.png)

后期使用的是`volatile state`;
