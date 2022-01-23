# 并发编程02-Java中的锁

# 1.Lock接口

Lock是一个接口,它定义了锁获取和释放的基本操作:

![image-20210511160628158](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511160628158.png)

* `void lock()`:获取锁,调用该方法当前线程将会获取锁,当锁获得后,从该方法返回;
* `void lockInterruptibly() throws InterruptedException`:可中断地获取锁,和`lock()`方法不同之处在于该方法会响应中断,即在锁的获取中可以中断当前线程;
* `boolean tryLock()`:尝试非阻塞的获取锁,调用该方法后立刻返回,如果能够获取则返回true,否则返回false;
* `boolean tryLock(long time.TimeUnit unit) throws InterruptedException`:超时的获取锁,当前线程在以下三种情况下会返回:
  * 当前线程在超时时间内获取了锁;
  * 当前线程在超时时间内被中断;
  * 超时时间结束,返回false;
* `void unlock`:释放锁;
* `Condition newCondition()`:获取等待通知组件,该组件与当前锁绑定,当前线程只有获取了锁,才能调用该组件的`wait()`方法,调用后,当前线程将释放锁;

# 2.队列同步器

## 2.1.AQS的关键方法

队列同步器AbstractQueuedSynchronizer是用来构建锁或者其他同步组件的基础框架.

重写同步器指定的方法时,需要使用同步器提供的如下3个方法来访问或修改同步状态:

1. `getState()`:获取当前同步状态;
2. `setState(int newState)`:设置当前同步状态;
3. `compareAndSetState(int expect,int update)`;使用CAS设置当前状态,该方法能够保证状态设置的原子性;

同步器可重写的方法:

* `boolean tryAcquire(int arg)`:独占式获取同步状态,实现该方法需要查询当前状态并判断同步状态是否符合预期,然后再进行CAS设置同步状态;
* `boolean tryRelease(int arg)`:独占式释放同步状态,等待获取同步状态的线程将有机会获取同步状态;
* `int tryAcquireShared(int arg)`:共享式获取同步状态,返回大于等于0的值,表示获取成功,反之获取失败;
* `boolean tryReleaseShared(int arg)`:共享式释放同步状态;
* `boolean isHeldExclusively()`:当前同步器是否在独占模式下被线程占用,一般该方法表示是否被当前线程所独占;

同步器的模板方法:

* `void acquire(int arg)`:独占式获取同步状态,如果当前线程获取同步状态成功,则由该方法返回,否则,将会进入同步队列等待,该方法将会调用重写的`tryAcquire(int arg)`方法;
* `void acquireInterruptibly(int arg)`:与`acquire(int arg)`相同,但是该方法响应中断,当前线程未获取到同步状态而进入到同步队列中,如果当前线程被中断,则该方法会抛出InterruptedException并返回;
* `boolean tryAcqureNanos(int arg,long nanos)`:在`acquireInterruptibly(int arg)`基础上增加了超时限制,如果当前线程在超时时间内没有获取到同步状态,那么将会返回false,如果获取到了返回true;
* `void acquireShared(int arg)`:共享式的获取同步状态,如果当前线程未获取到同步状态,将会进入同步队列等待,与独占式获取的主要区别是同一时刻可以有多个线程获取到同步状态;
* `void acquireSharedInterruptibly`:响应中断
* `void acquireSharedNanos(int arg,long nanos)`:增加了超时限制;
* `boolean release(int arg)`:独占式的释放同步状态;
* `boolean releaseShared(int arg)`:共享式的释放同步状态;
* `Coolection<Thread> getQueuedThreads()`:获取等待在同步队列上的线程集合;

## 2.2.AQS实现独占锁

如下是使用队列同步器快速实现独占锁

~~~java
package org.fechin.jmm;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * @Author:guoqing.zhu
 * @Date：2021/5/11 16:59
 * @Desription: TODO
 * @Version: 1.0
 */
public class Muatex implements Lock {

    //静态内部类,自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            if (getState() == 0) {
                throw new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
~~~

## 2.3.队列同步器的实现分析

### 2.3.1.同步队列

同步器依赖内部的同步队列(一个FIFO双向队列)来完成同步状态的管理,当前线程获取同步状态失败时,同步器会将当前线程以及等待状态等信息构造成一个节点(Node)并将其加入到同步队列,同时会阻塞当前线程,当同步状态释放,会把首节点中的线程唤醒,再次去尝试获取同步状态;

同步队列中的节点Node用来保存获取同步状态失败的线程引用,等待状态以及前驱和后继节点,节点属性类型如下:

* int waitStatus:等待状态:
  * CANCELLED:值为1,由于在同步队列中等待的线程等待超时或者被中断,需要从同步队列中取消等待,节点进入该状态将不会变化;
  * SIGNAL:值为-1,后继节点的线程处于等待状态,而当前节点的线程如果释放了同步状态或者被取消,将会通知后继节点,使得后继节点的线程得以运行;
  * CONDITION:值为-2,节点再等待队列中,节点线程在Conditon上,当其他线程对Condition调用了sign()方法后,该节点将会从等待队列中转移到同步队列中;
  * PROPAGTE:值为-3,表示下一次共享式同步状态将会无条件的被传播下去;
  * INITAL:值为0,初始状态;
* Node prev:前驱节点;
* Node next:后继节点;
* Node nextWaiter:等待队列中的后继节点.如果当前节点是共享的,那么这个字段是一个SHARED常量,也就是说节点类型(独占和共享)和等待队列中的后继节点公用同一个字段;
* Thread thread:获取同步状态的线程;

### 2.3.2.独占式同步状态获取和释放

#### 2.3.2.1.acquire

下面的代码主要完成了同步状态获取,节点构造,加入同步队列以及在同步队列中自旋.

首先调用自定义同步器实现的tryAcquire(int arg)方法,该方法保证线程安全的获取同步状态,如果同步状态获取失败,则构造同步节点(独占式Node.EXCLUSICE,同一时刻只能有一个线程成功获取同步状态)并通过`addWaiter(Node node)`方法将该节点加入到同步队列的尾部,最后调用`acquireQueued(Node node,int arg)`方法,使得该节点以死循环的方式获取同步状态,如果获取不到则阻塞节点中的线程,而被阻塞线程的唤醒主要依赖前驱节点的出队或阻塞线程被中断来实现.

![image-20210511182454158](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511182454158.png)

![image-20210511183038411](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511183038411.png)

![image-20210511183049555](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511183049555.png)

![image-20210511183440973](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511183440973.png)

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511183714638.png" alt="image-20210511183714638" style="zoom: 50%;" />

#### 2.3.2.2.release

![image-20210511183930917](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511183930917.png)

该方法执行时,会唤醒头节点的后继节点线程,`unparkSuccessor(Node node)`方法使用LockSupport来唤醒处于等待状态的线程.

### 2.3.3.共享式同步状态获取与释放

共享式访问资源时,其他共享式访问均被允许,而独占式访问被阻塞.独占式访问资源时候,同一时刻其他访问都被阻塞;

#### 2.3.3.1.acquire

![image-20210511184639212](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511184639212.png)

![image-20210511184606339](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511184606339.png)

#### 2.3.3.2.release

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511224902520.png" alt="image-20210511224902520" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511224919994.png" alt="image-20210511224919994" style="zoom:50%;" />

# 3.重入锁ReentrantLock

重入锁ReentrantLock顾名思义,就是支持重进入的锁,它表示该锁能够支持一个线程对资源的重复加锁,除此之外,该锁还支持获取锁时的公平和非公平选择.

## 3.1.实现重进入

1. 线程再次获取锁.锁需要去识别获取锁的线程是否为当前占据锁的线程,如果是,则再次成功获取;
2. 线程重复n次获取了锁,随后在第n次释放该锁后,其他线程能够获取到该锁.锁的最终释放要求锁对于获取进行计数新增,计数表示当前锁被重复获取的次数,而锁被释放时,计数自减,当计数等于0时表示锁已成功释放;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511230802781.png" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511230830907.png" alt="image-20210511230830907" style="zoom:50%;" />

该方法增加了再次获取同步状态的逻辑;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210511230958557.png" alt="image-20210511230958557" style="zoom:50%;" />

如果该锁被获取了n次,那么前(n-1)次`tryRelease()`方法必须返回false,而只有同步状态完全释放了,才能返回true;

## 3.2.公平与非公平锁的区别

![image-20210513173852846](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210513173852846.png)

公平的`tryAcquire(int acquires)`方法,加入了同步队列中当前节点是否有前驱节点的判断,如果该方法返回true.则表示有线程比当前线程更早地请求获取锁,因此需要等待前驱线程获取并释放锁之后才能继续获取锁.

# 4.读写锁ReentrantReadWriteLock

## 4.1.使用读写锁

在读操作时获取读锁,在写操作时获取写锁.当写锁被获取到时,后续的读写操作都会被阻塞,写锁释放之后,所有操作继续执行.

如下是使用ReentrantReadWriteLock和HashMap去实现缓存

~~~java
public class ReentrantReadWriteLockToBuildCache {

    static Map<String, Object> map = new HashMap<String, Object>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();

    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }

    public static final Object put(String key, Object value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
}
~~~

## 4.2.实现读写锁

### 4.2.1.读写状态分析

如果在一个整型变量上维护多种状态,就需要按位切割使用这个变量,读写锁将变量切分了两个部分,高16位表示读,低16位表示写.

### 4.2.2.写锁的获取与释放

![image-20210513180252755](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210513180252755.png)

写锁是一个支持重进入的排它锁,如果当前线程已经获取了写锁,则增加写状态.如果当前线程在获取写锁时,读锁已经被获取(读线程不为0)或者该线程不是已经获取写锁的线程,则当前线程进入等待状态.

写锁的释放与ReentrantLock一致;

### 4.2.3.读锁的获取与释放

读锁总会被成功地获取,而所做的也只是线程安全的增加读状态,如果当前线程已经获取了读锁,则增加读状态.如果当前线程在获取读锁时,则进入等待状态.从JDK5到JDK6后,增加了一些功能`getReadHoldCount()`

![image-20210513181024906](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210513181024906.png)

### 4.2.4.锁降级

未能理解

# 5.LockSupport工具

| 方法名称                        | 描述                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `void park()`                   | 阻塞当前线程,如果调用`unpark(Thread)`方法或者当前线程被中断,才能从`park()`方法返回; |
| `void parkNanos(long nanos)`    | 阻塞当前线程,最长不超过nanos纳秒,返回条件在`park()`的基础上增加了超时返回; |
| `void parkUntil(long deadline)` | 阻塞当前线程,直到deadline时间(从1970年到deadline时间的毫秒数); |
| `void unpark(Thread thread)`    | 唤醒处于阻塞状态的线程thread                                 |

# 6.Condition接口

任意一个Java对象都拥有一组监视器方法(定义在`java.lang.Object`上),主要包括`wait()`,`wait(long timeout)`,`notify()`以及`notifyAll()`,这些方法与`synchronized`同步关键字配合,可以实现等待/通知.Condition接口也提供了类似Object的监视器方法;

## 6.1.Condition接口与示例

~~~java
public class ConditionUseCase {

    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    public void conditionWait() throws InterruptedException {
        lock.lock();
        try {
            condition.await();
        } finally {
            lock.unlock();
        }
    }
    
    public void conditionSignal(){
        lock.lock();
        try {
            condition.signal();
        } finally {
            lock.unlock();
        }
    }
}
~~~

## 6.2.Condition的实现分析

ConditionObject是同步器AbstractQueuedSynchronizer的内部类.

### 6.2.1.等待队列

等待队列是一个FIFO的队列,在队列中的每个节点都包含了一个线程引用,该线程就是在Condition对象上等待的线程,如果一个线程调用`Condition.await()`方法,那么该线程将会释放锁,构造成节点加入等待队列并进入等待状态.事实上,节点的定义复用了同步节点的定义.也就是说,同步队列和等待队列中节点类型都是同步器的静态内部类AbstractQueuedSynchronizer.Node.

一个Condition包含一个等待队列,Condition拥有首节点(firstWaiter)和尾节点(lastWaiter).当前线程调用`Condition.await()`方法,将会以当前线程构造节点,并将节点从尾部加入等待队列.

在Object的监视器模型上,一个对象拥有一个同步队列和等待队列.而并发包中的Lock拥有一个同步队列和多个等待队列.如下图:

### 6.2.2.`await()`

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210513221854038.png" style="zoom: 50%;" />

从队列的角度看`await()`方法,当调用`await()`方法时,相当于同步队列的首节点(获取了锁的节点)移动到了Condition的等待队列中.

### 6.2.3.`signal()`

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210513224204028.png" alt="image-20210513224204028" style="zoom: 50%;" />

调用Condition的`signal()`方法,将会唤醒在等待队列的首节点,在唤醒节点之前将会将节点移动到同步队列.