# JVM详解11-JVM监控及诊断工具-GUI篇

# 1.工具概述

* JDK自带工具
  * **jconsole**:JDK自带的可视化监控工具.查看Java应用程序的运行情况,监控堆信息,永久区(或元空间)使用情况,类加载情况等;
  * **jvisualvm**:Visual VM是一个工具,它提供了一个可视化界面,用于查看Java虚拟机运行的基于Java技术应用程序的详细信息;
  * **jmc**:Java Mission Control,内置Java Flight Recorder,能够以极低的性能开销收集Java虚拟机的性能数据;
* 第三方工具
  * **MAT**:Memory Analyzer Tool基于Eclipse的内存分析工具,是快速,功能丰富的Java heap分析工具,可以帮助查找内存泄漏和减少内存消耗;
  * **JPofiler**:商业软件,需要付费,功能强大;
  * **Arthas**:Alibaba开源的Java诊断工具;
  * **Btrace**:Java运行时追踪工具,可以在不停机的情况下,跟踪指定的方法调用,构造函数调用和系统内存等信息;

# 2.JConsole

JConsole比较简陋,jvisualvm可以完全替代JConsole;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211104142913600.png" alt="image-20211104142913600" style="zoom:67%;" />

# 3.VisualVM

## 3.1.Visual VM插件

* 可以在可用插件页面下,在线安装插件;
* http://visualvm.github.io/pluginscenters.html离线下载插件文件,然后在Plugin对话框的已下载页面添加已下载插件;

## 3.2.远程连接

1. 确定远程服务器的ip地址;
2. 添加JMX(通过JMX技术具体监控远端服务器哪个Java进程);
3. 修改bin/catalina.sh文件,连接远程的tomcat;
4. 在../conf中添加jmxremote.access和jmxremote.password文件;
5. JMX中输入端口号,用户名,密码登录;

# 4.Memory Analyzer Tool

## 4.1.MAT简单功能概述

MAT是功能强大的Java堆内存分析器,**主要用于分析内存dump文件,查找内存泄漏以及查看内存消耗情况**;

* Histogram:展示各个类的实例数目以及这些实例的ShallowHeap和RetainedHeap;

![image-20211108100629519](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211108100629519.png)

* Thread overview:查看系统中的Java线程和查看局部变量的信息

* outgoing incoming references:获取对象相互引用的关系

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211108100918544.png)

## 4.2.shallow heap和retained heap:潜堆和深堆

* shallow heap:以String为例,2个int值共占8个字节,对象引用占用4个字节,对象头8个字节合集20字节,向8字节对齐,故占用24字节(32位系统JDK7).这24个字节为String对象的潜堆大小,它与String的value实际取值无关,无论字符串长度如何,浅堆大小始终是24字节;
* retained heap
  * Retained Set(保留集)对象A的保留集指当对象A被垃圾回收后,可以被释放的所有的对象集合(包括对象A本身),即对象A的保留集可以被认为是只能通过对象A被字节或间接访问到的所有对象的集合.通俗地说,就是指仅被对象A所持有的对象的集合;
  * Retained Heap(深堆):深堆是指对象的保留集中所有的对象的浅堆大小之后;
* 区别:浅堆指的是对象本身占用的内存,不包括其内部引用对象的大小,一个对象的深堆只能通过该对象访问到的(直接或间接)所有对象的浅堆之和,即对象被回收后,可以释放的真实空间;

## 4.3.Domainator Tree:支配树

![image-20211108105906005](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211108105906005.png)

MAT提供了一个称为支配树的对象图,支配树体现了对象实例之间的支配关系.在对象引用图中:

* 所有指向对象B的路径都经过对象A,那么,**对象A支配对象B**;
* 如果对象A是离对象B最近的一个支配对象,则认为**对象A为对象B的直接支配者**.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211108112413138.png" alt="image-20211108112413138" style="zoom:50%;" />



左侧为对象的引用图,右侧为支配树;

# 5.JProfiler

https://www.ej-technologies.com/products/jprofiler/docs?redirect=true

# 6.Arthas

https://github.com/alibaba/arthas/blob/master/README_CN.md

https://arthas.aliyun.com/doc/

![image-20211108211322875](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211108211322875.png)

根据官网手册,启动`arthas-boot.jar`来监测`math-game.jar`;

![image-20211108213224517](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211108213224517.png)

# 7.Java Mission Control

mac无法打开,具体内容查看<深入理解Java虚拟机 第三版>171页;

# 8.内存泄漏案列分析

## 8.1.内存泄漏的理解与分类

可达性分析算法来判断对象是否是不再使用的对象,本质上都是判断一个对象是否还被使用,那么对于这种情况下,由于代码的实现不同就会出现多种内存泄漏问题(让JVM误以为此对象还在引用中,无法回收,造成内存泄漏);

* 狭义:严格来说,只有对象不再被程序用到了,但是GC又不能回收它们的情况,才叫内存泄漏;
* 广义:很多一些不太好的实践会导致对象的生命周期变得很长甚至导致OOM,可以叫做宽泛意义上的内存泄漏;

## 8.2.内存泄漏的情况

* **静态集合类**:如果把类似于HashMap,ArrayList等容器.如果这些容器为静态的,那么它们的生命周期与JVM程序一致.则容器中的对象在程序结束之前将不能被释放.从而造成内存泄漏;

```java
public class MemoryLeak{
  static List list = new ArrayList();
  
  public void oomTests(Object obj){
    list.add(obj);
  }
}
```

* **单例模式**:与静态集合导致内存泄漏的原因类似,它的生命周期和JVM的生命周期一样长,所以如果单例对象如果持有外部对象的引用,那么这个外部对象也不会被回收,那么就会造成内存泄漏;
* **内部类持有外部类**:如果一个外部类的实例对象的方法返回了一个内部类的实例对象,这个内部类对象被长期引用,外部类对象也不会被垃圾回收;
* **各种连接,如数据库连接,网络连接,IO连接**:在对数据库进程操作的过程中,首先要简历与数据库的连接,当不再使用时,需要调用close方法来进行释放与数据库的连接,只有连接被关闭,垃圾回收器才会回收相应的资源;
* **变量不合理的作用域**:一般而言,一个变量的定义的作用范围大于其使用范围,很有可能会造成内存泄漏.另一方面,如果没有及时地把对象设置为null,很有可能导致内存泄漏的发生;

```java
public class UsingRandom{
  private String msg;
  public void receiveMsg(){
    msg = readFromNet();//从网络中接收数据保存到msg;
    saveDB(msg);//把msg保存到数据库中;
  }
}
```

msg的生命周期与对象的生命周期相同,此时msg还不能回收,因此造成了内存泄漏;

* **改变哈希值**:当一个对象被存储在HashSet等容器中之后,就不能修改这个对象中那些茶语计算的哈希值的字段了.否则修改后哈希值就与最初存进去的不一致,在这种情况下,这导致无法从HashSet集合中单独删除当前对象,导致内存泄漏;

* **缓存泄漏**:缓存长时间使用就会造成大量的无用数据,实际上也是一种泄漏,可以使用weak引用或者LRU算法;
* **监听器和回调**:如果客户端在实现的API中注册回调,却没有显示的取消,那么就会积聚.可以使用弱引用;

## 8.3.案例分析

~~~java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) { //入栈
        ensureCapacity();
        elements[size++] = e;
    }
    //存在内存泄漏
//    public Object pop() { //出栈
//        if (size == 0)
//            throw new EmptyStackException();
//        return elements[--size];
//    }

    //正确的使用方法
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null;
        return result;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
~~~

# 9.OQL语句

可以参考<深入立即Java虚拟机的附录D>;

## 9.1.SELECT

* `SELECT * FROM`结果查看对象的引用实例(相当于outgoing references);
* `SELECT OBJECTS FROM`  结果返回结果集中的项以对象的形式显示;
* `SELECT AS RETAINED SET * FROM`:可以得到所有对象的保留集;
* `SELECT DISTINCT OBJECTS`:用于在结果集中去除重复对象;

## 9.2.FROM

FROM子句用于指定查询范围,可以指定类名,正则表达式或者对象地址;

* `SELECT * FROM java.lang.String s`;
* `SELECT * FROM "com\.fechin\..*"`;正则表达式,输出所有com.fechin包下所有类的实例;
* `SELECT * FROM 0x37a0b4d`;直接使用类的地址;

## 9.3.WHERE

* `SELECT * FROM char[] s WHERE s.@length>10`;
* `SELECT * FROM java.lang.String s WHERE toString(s) LIKE ".*java.*"`;
* `SELECT * FROM java.util.Vector v WHERE v.elementData.@length>15 AND v.@retainedHeapSize>1000`;返回数组长度大于15,并且深堆大于1000字节的所有Vector对象;



