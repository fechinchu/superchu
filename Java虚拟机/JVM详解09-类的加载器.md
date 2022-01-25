# JVM详解09-类的加载器

# 1.类加载器概述

## 1.1.ClassLoader的作用

ClassLoader是Java的核心组件,所有的Class都是由ClassLoader进行加载的,**ClassLoader负责通过各种方式将Class信息的二进制数据流读入JVM内部,转换为一个与目标对应的`java.lang.Class`对象实例.**然后交给Java虚拟机进行链接,初始化等操作.因此,ClassLoader在整个装载阶段,只能影响到类的加载,而无法通过ClassLoader去改变类的链接和初始化行为.至于它是否可以运行,则由Execution Engine决定;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211029192909913.png" alt="image-20211029192909913" style="zoom: 33%;" />

## 1.2.类加载器的命名空间

* 类的唯一性:
  * 对于任意一个类,都需要由加载它的类加载器和这个类本身一起确认其在Java虚拟机中的唯一性.每一个类加载器,都拥有一个独立的类名称空间,比较两个类是否相等,只有在这两个类是由同一个类加载器加载的前提下才有意义.否则,及时这两个类源自同一个Class文件,被同一个虚拟机加载,只要加载他们的类加载器不同,那这两个类必定不相等;
* 命名空间:
  * 每个类加载器都有自己的命名空间,命名空间由该加载器及所有的父加载器所加载的类组成;
  * 在同一命名空间中,不会出现类的完整名字相同的两个类;但是在不同的命名空间中,有可能;
  * 在应用中,往往借助这一特性,来运行同一个类的不同版本;

## 1.3.类加载器的基本特性

* 双亲委派模型.但不是所有类加载都遵守这个模型,有的时候,启动类加载器所加载的类型,是可能要加载用户代码的,比如JDK内部的ServiceProvider/ServiceLoader机制,用户可以在标准API框架上,提供自己的实现,JDK也需要提供默认的参考实现.比如,Java中JNDI,JDBC,文件系统,Cipher等很多方面,都是利用这种机制,这种情况就不会用双亲委派模型去加载,而是利用所谓的上下文加载器;
* 可见性,子类加载器可以访问父加载器加载的类型,但是反过来不允许.不然,因为缺少必要的隔离,就没有办法利用类加载器去实现容器的逻辑;
* 单一性,由于父加载器的类型对于子加载器是可见的,所以父加载器中加载过的类型,就不会在子加载器重复加载.

# 2.类的加载器

## 2.1.类的加载器概述

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211031155628793.png" alt="image-20211031155628793" style="zoom:50%;" />

* 除了顶层的启动类加载器外,其余的类加载器都应该有自己的"父类"加载器;
* 不同类的加载器看是继承关系,实际是包含关系.在下层加载器中,包含着上层加载器的引用;

## 2.2.引导类加载器

- 这个类加载使用C/C++语言实现的,嵌套在JVM内部;
- 它用来加载Java的核心库(JAVA_HOME/jre/lib/rt.jar,resources.jar,sun.boot.class.path路径下的内容),用于提供JVM自身需要的类;
- 并不继承java.lang.ClassLoader(其余两种类加载器都继承于它),没有父加载器;
- 引导类加载器加载扩展类和应用程序类加载器,并指定为他们的父类加载器;
- 出于安全考虑,Bootstrap启动类加载器只加载包含名为java,javax,sun等开头的类;
- 加载扩展类和引用程序类加载器,并指定为他们的父类加载器;

## 2.4.扩展类加载器

- Java语言编写,`sun.misc.Launcher$ExtClassLoader`实现;
- 继承于ClassLoader类;
- 父类加载器为启动类加载器;
- 从`java.ext.dirs`系统属性所指定的目录中加载类库,或从JDK的安装目录`jre/lib/ext`子目录(扩展目录)下加载类库,如果用户创建JAR放在此目录下,也会自动由扩展类加载器加载;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211031161104708.png" alt="image-20211031161104708" style="zoom: 67%;" />

## 2.5.应用类加载器

- Java语言编写,`sun.misc.Launcher$AppClassLoader@18b4aac2`实现;
- 继承于ClassLoader类;
- 父类加载器为扩展类加载器;
- 它负责加载环境变量classpath或系统属性`java.class.path`指定路径下的类库;
- 该类加载是程序中默认的类加载器;
- 它是用户自定义类加载器的默认父加载器;

## 2.6.用户自定义类加载器

* 自定义类加载器可以实现插件机制.例如:OSGI组件框架,Eclipse的插件机制;
* 自定义类加载器可以实现应用隔离,例如:Tomcat,Spring等中间件和组件框架都在内部实现了自定义的加载器,并通过自定义加载器隔离不同的组件模块;
* 自定义类加载器通常要继承于ClassLoader;

```java
public class MyClassLoader extends ClassLoader {
    private String byteCodePath;

    public MyClassLoader(String byteCodePath) {
        this.byteCodePath = byteCodePath;
    }

    public MyClassLoader(ClassLoader parent, String byteCodePath) {
        super(parent);
        this.byteCodePath = byteCodePath;
    }

    @Override
    protected Class<?> findClass(String className) throws ClassNotFoundException {
        BufferedInputStream bis = null;
        ByteArrayOutputStream baos = null;
        try {
            //获取字节码文件的完整路径
            String fileName = byteCodePath + className + ".class";
            //获取一个输入流
            bis = new BufferedInputStream(new FileInputStream(fileName));
            //获取一个输出流
            baos = new ByteArrayOutputStream();
            //具体读入数据并写出的过程
            int len;
            byte[] data = new byte[1024];
            while ((len = bis.read(data)) != -1) {
                baos.write(data, 0, len);
            }
            //获取内存中的完整的字节数组的数据
            byte[] byteCodes = baos.toByteArray();
            //调用defineClass()，将字节数组的数据转换为Class的实例。
            Class clazz = defineClass(null, byteCodes, 0, byteCodes.length);
            return clazz;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (baos != null)
                    baos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                if (bis != null)
                    bis.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return null;
    }
}

public class MyClassLoaderTest {
    public static void main(String[] args) {
        MyClassLoader loader = new MyClassLoader("d:/");

        try {
            Class clazz = loader.loadClass("Demo1");
            System.out.println("加载此类的类的加载器为：" + clazz.getClassLoader().getClass().getName());

            System.out.println("加载当前Demo1类的类的加载器的父类加载器为：" + clazz.getClassLoader().getParent().getClass().getName());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

# 3.ClassLoader源码分析

## 3.1.ClassLoader与类加载器的关系

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211031170548838.png" alt="image-20211031170548838" style="zoom:50%;" />

除了以上虚拟机自带的加载器外,用户可以定制柜自己的类加载器.Java提供了抽象类`java.lang.ClassLoader`所有用户自定义的类加载器都应该继承ClassLoader;

## 3.2.Launcher类

Launcher主要被系统用于启动主应用程序;

如下是创建扩展类加载器和应用类加载器的源码;

![image-20211031172245891](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211031172245891.png)

## 3.3.ClassLoader类主要方法

* `getParent()`:返回该类加载器的父类加载器;
* `loadClass(String name)`:加载名为name的类,返回结果为`java.lang.Class`类的实例.如果找不到类,则返回ClassNotFoundException异常,该方法中的逻辑就是**双亲委派模式的实现**;(源码中是用递归实现);

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211031173743614.png" alt="image-20211031173743614" style="zoom:50%;" />

* `protected Class findClass(String name)`:查找二进制名称为name的类,返回结果为`Java.lang.Class`类的实例.这是一个受保护的方法,JVM推荐我们重写此方法,需要自定义加载器遵循双亲委派机制.该方法会在检查完父类加载器之后被`loadClass()`方法调用;

  * 在JDK1.2之前,在自定义类加载时,总会去继承ClassLoader类并重写loadClass方法,从而实现自定义的类加载类.但是在JDK1.2之后已不再建议用户去覆盖`loadClass()`方法,而是建议把自定义的类加载逻辑写在`findClass()`中,`findClass()`方法是在`loadClass()`中被调用的,当`loadClass()`方法中加载器加载失败后,则会调用自己的`findClass()`来完成类的加载;

  * ![image-20211031181316590](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211031181316590.png)

    **ClassLoader类中并没有实现`findClass()`方法的具体代码,并且,`findClass()`通常适合`defineClass()`一起使用的.一般情况下,在自定义加载器时,会直接覆盖`ClassLoader`的`findClass()`方法并编写加载规则,取得要加载类的字节码后转换成流,然后调用`defineClass()`方法生成类的Class对象**;

* `protected Class defineClass(String name,byte[] b,int off,int len);`根据给定的字节数组b转换成Class的实例,off和len参数表示实际Class信息在byte数组中的位置和长度,其中byte数组b是ClassLoader从外部获取的.

  * `defineClass()`方法是用来将byte字节流解析成JVM能够识别的Class对象(ClassLoader中已经实现该方法逻辑);
  * **`defineClass()`方法通常与`findClass()`方法一起使用,一般情况下,在自定义类加载器时,会直接覆盖ClassLoader的`findClass()`方法并编写加载规则,取得要加载类的字节码转换成流,然后调用`defineClass()`方法生成类的Class对象;**

* `resolveClass(Class c)`;

  * 链接指定一个Java类.使用该方法可以使用类的Class对象创建完成的同时也被解析;

* `findLoadedClass(String name)`;

  * 查找名称为name的已经被加载过的类,返回结果为`java.lang.Class`类的实例;

* `private find ClassLoader parent`;

  * 它也是一个ClassLoader的实例,这个字段所表示的ClassLoader也称为这个ClassLoader的双亲;在类加载的过程中,ClassLoader可能会将某些请求交于自己的双亲处理;

## 3.4.SecureClassLoader与URLClassLoader

* SecureClassLoader扩展了ClassLoader,新增了几个与使用相关的代码源(对代码源的位置及其证书的验证)和权限定义类验证(主要指对class源码的访问权限)的方法,一般不会直接与这个类打交道;
* URLClassLoader这个实现类为ClassLoader的很多方法提供了具体的实现,并新增了URLClassPath类协助获得Class字节码流功能.**在编写自定义类加载器时,如果没有太过于复杂的需求,可以直接继承URLClassLoader**;

## 3.5.ExtClassLoader与AppClassLoader

* ExtClassLoader与AppClassLoader都继承于URLClassLoader;
* ExtClassLoader并没有重写`loadClass()`方法,AppClassLoader重载了`loadClass()`方法,但最终还是调用的父类的`loadClass()`,因此依然遵守双亲委派机制;

## 3.6.`Class.forName()`与`ClassLoader.loadClass()`;

* `Class.forName()`是一个静态方法,最常用的是`Class.forName(String className)`;根据传入的类的全限定名返回一个Class对象;**该方法在将Class文件加载到内存的同时,会执行类的初始化**;
* `ClassLoader.loadClass()`是一个实例方法.**该方法将Class文件加载到内存时,并不会执行类的初始化,直到这个类第一次使用时才进行初始化**;

# 4.双亲委派机制

## 4.1.双亲委派模型的优势

* 避免类的重复加载,确保一个类的全局唯一性;
* 保护程序安全,防止核心API被随意篡改;

## 4.2.双亲委派模型的劣势

* 检查类是否加载的委托过程是单向的,这个方式虽然从结构上比较清晰,使各个ClassLoader的职责非常明确,但会导致顶层的ClassLoader无法访问底层的ClassLoader所加载的类;

## 4.3.破坏双亲委派机制

### 4.3.1.第一次破坏双亲委派机制

双亲委派模型在JDK1.2才被引入,但是类加载器的概念和ClassLoader在1.0就已经存在,为了兼容,只能在1.2之后,在`java.lang.ClassLoader`中添加一个protected方法`findClass()`,并引导用户编写的类加载逻辑时尽可能去重写这个方法,而不是在`loadClass`中编写代码;

### 4.3.2.第二次破坏双亲委派机制-线程上下文类加载器

双亲委派模型的第二次"被破坏"是由这个模型自身的缺陷导致的,双亲委派很好地解决了各个类加载器协作时基础类型的一致性问题(越基础的类由越上层的加载器进行加载);如果有基础类型要调回用户的代码.就会出现问题;

![image-20211101110704241](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211101110704241.png)

典型例子就是JNDI服务,JNDI是Java的标准服务,它的代码有启动类加载完成,但JNDI存在的目的就是对资源进行查找和集中管理,它需要其他厂商实现并部署在应用程序的ClassPath下的JNDI服务提供者接口(SPI)的代码,Java的设计团队引入了线程上下文类加载器.JNDI服务使用这个线程上下文类加载器去加载所需的SPI服务代码;在JDK1.6时候,JDK提供了`java.util.ServiceLoader`以`META-INF/services`中的配置信息,用责任链模式,这才是给SPI的加载提供了合理的解决方案;

### 4.3.3.第三次破坏双亲委派机制-代码热替换,模块热部署

JSR-291(OSGi R4.2)实现模块化热部署的关键是它自定义的类加载器机制.在OSGi环境下,类加载器不再双亲委派模型的树状结构,而是发展为复杂的网状结构;

# 5.沙箱安全机制

Java安全模型的核心就是Java沙箱(sandbox),沙箱机制就是**将Java代码限定在虚拟机(JVM)特定的运行范围中,并且严格限制代码对本地系统资源访问**;

## 5.1.JDK1.0时期

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211101113631362.png" style="zoom: 33%;" />

在Java中将执行程序分层本地代码和远程代码两种,本地代码默认为可信任的,远程代码看作不受信任的;本地代码可以访问一切本地资源,远程代码完全依赖于沙箱;

## 5.2.JDK1.1时期

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211101113943196.png" alt="image-20211101113943196" style="zoom:33%;" />

Java1.1针对安全机制做了改进,增加了安全策略;

## 5.3.JDK1.2时期

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211101114030821.png" alt="image-20211101114030821" style="zoom:33%;" />

## 5.4.JDK1.6时期

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211101114151051.png" alt="image-20211101114151051" style="zoom:33%;" />

虚拟机把所有代码加载到不同的系统域和应用域中,系统域部分专门负责与关键资源进行交互.而各个应用域部分则通过系统域的部分代理来对各种需要的资源进行访问.虚拟机中不同的受保护域,对应不一样的权限.存在于不同域中的类文件就具有了当前域的全部权限;

