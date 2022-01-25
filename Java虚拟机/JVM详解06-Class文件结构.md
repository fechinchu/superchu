# JVM详解06-Class文件结构

# 1.字节码文件简介

## 1.1.文档规范

Java虚拟机规范文档链接:

https://docs.oracle.com/javase/specs/index.html

![image-20211026171425387](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211026171425387.png)

![image-20211026171555329](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211026171555329.png)

## 1.2.前端编译器

* 想要让一个Java程序正确地运行在JVM中,Java源码就必须要被编译为符合JVM规范的字节码;

  * **前端编译器**主要任务就是负责将符合Java语法规范的Java代码转换为符合JVM规范的字节码文件;
  * javac就是一种能够将Java源码编译为字节码的前端编译器;
  * javac编译器在将Java源码编译为一个有效的字节码文件过程中经历了4个步骤:
    * 词法解析;
    * 语法解析;
    * 语义解析;
    * 生成字节码;

  ![image-20211026172853213](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211026172853213.png)

HotSpot VM并没有强制要求前端编译器只能使用javac来编译字节码,其实只要编译结果符合JVM规范都可以被JVM识别即可,在Java的前端编译器领域,除了javac之外,还有内置在Eclipse的ECJ(Eclipse Complier for Java);

* javac是全量编译;(IDEA默认javac编译,还可以自己设置);
* ECJ是增量编译;

**前端编译器并不会直接涉及编译优化等方面的技术,而是将这些具体优化细节交给HotSpot的JIT编译器负责**;

## 1.3.使用字节码文件解决的问题

~~~java
public class IntegerTest {
    public static void main(String[] args) {

        Integer x = 5;
        int y = 5;
        System.out.println(x == y);//true

        Integer i1 = 10;
        Integer i2 = 10;
        System.out.println(i1 == i2);//true

        Integer i3 = 128;
        Integer i4 = 128;
        System.out.println(i3 == i4);//false

    }
}
~~~

如果单独看代码是不能理解原因的,这时候可以从字节码入手:`javap -v IntegerTest`;

![image-20211026180417531](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211026180417531.png)

## 1.4.字节码文件概述

* 字节码文件中是什么?
  * 源代码通过编译器之后便会生成一个字节码文件,字节码是一种二进制的类文件,它的内容是JVM指令.并不像C,C++由编译器直接生成机器码;
* 什么是字节码指令(byte code)?
  * Java虚拟机的指令由一个直接长度的,代表着某种特定操作含义的操作码(opcode)以及跟随其后的零至多个代表此操作所需要参数的操作数(operand)所构成.虚拟机中许多指令并不包含操作数,只有一个操作码;

* Class类的本质
  * 任何一个Class文件都对应着唯一一个类或接口的定义信息,但反过来说,Class文件实际上并不一定以磁盘文件的形式存在.Class文件是一组以8位字节为基础单位的二进制流;
* Class文件格式
  * Class的文件结构没有任何分隔符号.所以在其中的数据项,无论是字节顺序还是数量,都是被严格限定的,哪个字节代表什么含义,长度多少,先后顺序如何,都不允许改变;
  * Class的文件格式采用一种类似于C语言结构体的方式进行数据存储,这种结构中只有两种数据类型:**无符号数**和**表**;
    * 无符号数属于基本的数据类型,以u1,u2,u4,u8来代表1个字节,2个字节,4个字节,8个字节的无符号数,无符号数可以用来描述数字,索引引用,数量值或者按照UTF-8编码构成字符串值;
    * 表是由多个无符号数或者其他表作为数据项构成的符合数据类型,所有表都习惯性地以"_info"结尾,表用于描述有层次关系的符合结构的数据,整个Class文件本质上就是一张表.由于没有固定长度,所以通常会在其前面加上个数说明;

# 2.字节码文件结构

## 2.1.结构概述

Class文件的结构并不是一成不变的,但是其基本结构和框架是非常稳定的;

* 魔数;
* Class文件版本;
* 常量池;
* 访问标志;
* 类索引,父类索引,接口索引集合;
* 字段表集合;
* 方法表集合;
* 属性表集合;

![image-20211027101908908](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027101908908.png)

| 类型           | 名称                | 说明                   | 长度    | 数量                  |
| -------------- | ------------------- | ---------------------- | ------- | --------------------- |
| u4             | magic               | 魔数,识别Class文件格式 | 4个字节 | 1                     |
| u2             | minor_version       | 副版本号(小版本)       | 2个字节 | 1                     |
| u2             | major_version       | 主版本号(大版本)       | 2个字节 | 1                     |
| u2             | constant_pool_count | 常量池计数器           | 2个字节 | 1                     |
| cp_info        | constant_pool       | 常量池表               | n个字节 | constant_pool_count-1 |
| u2             | access_flags        | 访问标识               | 2个字节 | 1                     |
| u2             | this_class          | 类索引                 | 2个字节 | 1                     |
| u2             | super_class         | 父类索引               | 2个字节 | 1                     |
| u2             | interfaces_count    | 接口计数器             | 2个字节 | 1                     |
| u2             | interfaces          | 接口索引集合           | 2个字节 | interfaces_count      |
| u2             | fields_count        | 字段计数器             | 2个字节 | 1                     |
| field_info     | fields              | 字段表                 | n个字节 | fields_count          |
| u2             | methods_count       | 方法计数器             | 2个字节 | 1                     |
| method_info    | methods             | 方法表                 | n个字节 | methods_count         |
| u2             | attributes_count    | 属性计数器             | 2个字节 | 1                     |
| attribute_info | attributes          | 属性表                 | n个字节 | attributes_count      |

![image-20211027150736501](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027150736501.png)

## 2.2.魔数

唯一作用是确定这个文件是否为一个能被虚拟机接收的有效合法的Class文件.魔数是Class文件的标识符;

如果Class文件不以`0xCAFFBABE`开头,虚拟机在进行文件校验时就会抛出`Java.lang.ClassFormatError:Incompatible magic vlaue xxxxxx in class file xxxxx`;

## 2.3.版本号

* 第5个和第6个直接代表的含义是编译的副版本号minor_version,而第7个和第8个直接就是编译的主版本号major_version;
* 它们构成了class文件的版本号;版本号和Java编译器的对应关系如下:

| 主版本（十进制） | 副版本（十进制） | 编译器版本 |
| ---------------- | ---------------- | ---------- |
| 45               | 3                | 1.1        |
| 46               | 0                | 1.2        |
| 47               | 0                | 1.3        |
| 48               | 0                | 1.4        |
| 49               | 0                | 1.5        |
| 50               | 0                | 1.6        |
| 51               | 0                | 1.7        |
| 52               | 0                | 1.8        |
| 53               | 0                | 1.9        |
| 54               | 0                | 1.10       |
| 55               | 0                | 1.11       |

* 不同版本的Java编译器编译的Class文件对应的版本是不一样的.目前,高版本的Java虚拟机可以执行由低版本编译器生成的Class文件,但是低版本的Java虚拟机不能执行由高版本编译器生成的Class文件,否则JVM会抛出`java.lang.UnsuppportedClassVersionError`异常;

## 2.4.常量池

* 在版本号之后,紧跟着的是常量池的数量,以及若干个常量池表;
* 常量池中常量的数量是不固定的,所以在常量池的入口需要放置一项u2类型的无符号数,代表常量池容量计数值(constant_pool_count),这个容量计数是从1开始;
* Class文件使用了一个前置的容量计数器(constant_pool_count)加上若干连续的数据项(constant_pool)的形式来描述常量池内容,这一系列连续常量池数据为常量池集合;
* 常量池表中,用于存放编译期生成的各种字面量和符号引用,这部分内容将在类加载进入方法区的**运行时常量池**中存放;

![image-20211027153015118](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027153015118.png)

### 2.4.1.常量池计数器

常量池容器记数值(u2类型):**从1开始**,表示常量池中有多少项常量.即`constant_pool_count=1`表示常量池中有0个常量项;

### 2.4.2.常量池表

* 常量池主要存放两大类常量:**字面量**和**符号引用**;
* 常量池表包含了class文件结构及其子结构中引用的所有字符串常量,类或接口名,字段名和其他常量.
* 常量池中的每一项都具备相同的特征:第一个直接作为类型标记,用于确定该项的格式,这个字节称为tag type;

| 类型                             | 标志(或标识) | 描述                   |
| -------------------------------- | ------------ | ---------------------- |
| CONSTANT_utf8_info               | 1            | UTF-8编码的字符串      |
| CONSTANT_Integer_info            | 3            | 整型字面量             |
| CONSTANT_Float_info              | 4            | 浮点型字面量           |
| CONSTANT_Long_info               | 5            | 长整型字面量           |
| CONSTANT_Double_info             | 6            | 双精度浮点型字面量     |
| CONSTANT_Class_info              | 7            | 类或接口的符号引用     |
| CONSTANT_String_info             | 8            | 字符串类型字面量       |
| CONSTANT_Fieldref_info           | 9            | 字段的符号引用         |
| CONSTANT_Methodref_info          | 10           | 类中方法的符号引用     |
| CONSTANT_InterfaceMethodref_info | 11           | 接口中方法的符号引用   |
| CONSTANT_NameAndType_info        | 12           | 字段或方法的符号引用   |
| CONSTANT_MethodHandle_info       | 15           | 表示方法句柄           |
| CONSTANT_MethodType_info         | 16           | 标志方法类型           |
| CONSTANT_InvokeDynamic_info      | 18           | 表示一个动态方法调用点 |

#### 2.4.2.1.字面量和符号引用

* 字面量包括:

  * 文本字符串:`String str = "fechin"`;
  * 声明为final的常量值:`final int NUM = 10`

* 符号引用包括:

  * 类和接口的全限定名:`com/fechin/test/Demo`
  * 字段的名称和描述符:
    * 名称:`add()`方法的名称就是add;
    * 描述符:描述符的作用是用来描述字段的数据类型,方法的参数列表(包括数量,类型以及顺序)和返回值.根据描述符的规则,基本数据类型以及void类型都用一个大写字符来表示.对象类型用字符L加对象的全限定名来表述;

  | 标志符 | 含义                                                 |
  | ------ | ---------------------------------------------------- |
  | B      | 基本数据类型byte                                     |
  | C      | 基本数据类型char                                     |
  | D      | 基本数据类型double                                   |
  | F      | 基本数据类型float                                    |
  | I      | 基本数据类型int                                      |
  | J      | 基本数据类型long                                     |
  | S      | 基本数据类型short                                    |
  | Z      | 基本数据类型boolean                                  |
  | V      | 代表void类型                                         |
  | L      | 对象类型，比如：`Ljava/lang/Object;`                 |
  | [      | 数组类型，代表一维数组。比如：`double[][][] is [[[D` |

  * 方法的名称和描述符;

> 说明:
>
> * 虚拟机在加载Class文件时才会进行动态链接,也就是说Class文件中不会保存各个方法和字段的最终内容布局信息,因此,这些字段和方法的符号引用不经过转换是无法直接被虚拟机使用的.**当虚拟机运行时,需要从常量池中获得对应的符号引用,再在类加载过程中的解析阶段将其替换为直接引用,并翻译到具体的内存地址中**;
> * 符号引用:符号引用以一组符号来描述所引用的目标,符号可以是任何形式的字面量,只要使用时能无歧义的定位到目标即可;符号引用与虚拟机实现的内存布局无关,引用的目标并不一定加载到内存中;
> * 直接引用:直接引用可以是直接指向目标指针,相对偏移量或者是一个能间接定位到目标的句柄,直接引用是与虚拟机实现的内存布局相关的.同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同.如果有直接引用,那说明引用的目标一定已经存在于内存中了;

#### 2.4.2.2.常量类型和结构

![image-20211027164114983](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027164114983.png)

## 2.5.访问标识

| 标志名称       | 标志值 | 含义                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 标志为public类型                                             |
| ACC_FINAL      | 0x0010 | 标志被声明为final，只有类可以设置                            |
| ACC_SUPER      | 0x0020 | 标志允许使用invokespecial字节码指令的新语义，JDK1.0.2之后编译出来的类的这个标志默认为真。（使用增强的方法调用父类方法） |
| ACC_INTERFACE  | 0x0200 | 标志这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract类型，对于接口或者抽象类来说，次标志值为真，其他类型为假 |
| ACC_SYNTHETIC  | 0x1000 | 标志此类并非由用户代码产生（即：由编译器产生的类，没有源码对应） |
| ACC_ANNOTATION | 0x2000 | 标志这是一个注解                                             |
| ACC_ENUM       | 0x4000 | 标志这是一个枚举                                             |

访问标记用两个字节标识,用于识别一些类或者接口层次的访问信息.包括这个Class是类还是接口;是否定义为public类型,是否定义为abstract类型;

## 2.6.类索引,父类索引,接口索引集合

| 长度 | 含义                         |
| ---- | ---------------------------- |
| u2   | this_class                   |
| u2   | super_class                  |
| u2   | interfaces_count             |
| u2   | interfaces[interfaces_count] |

* 这三项数据来确定这个类的继承关系;
  * 类索引用于确定这个类的全限定名;
  * 父类索引用于确定这个类的父类的全限定名.由于Java语言不允许多重继承,所以父类索引只有一个,除了`Object`外,所有Java类的父类索引都不为0;
  * 接口索引集合用来描述这个类实现了哪些接口;

## 2.7.字段表集合

 ### 2.7.1.字段计数器

`fields_count`的值表示当前class文件fields表的成员个数.使用两个字节来表示;

### 2.7.2.字段表

* 字段表中的字段用于描述接口或类声明的变量,字段(field)包括类级变量以及实例级变量,但是不包括方法内部,代码块内部声明的变量;
* 字段表集合中不会出现父类或者实现的接口中的字段,但是有可能列出Java代码中不存在的字段,比如在内部类中为了保持对外部类的访问性,会自动添加指向外部类实例的字段;
* 字段叫什么名字,字段被定义成什么数据类型,这些都是无法固定的,只能引用常量池中的常量进行描述;
* 它指向常量池索引集合,他描述了每个每个字段的完整信息.比如字段的标识符,访问修饰符,是类变量还是实例变量,是否是final修饰的常量等;

字段表的结构如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027203838839.png" alt="image-20211027203838839" style="zoom: 67%;" />

#### 2.7.2.1.字段表访问标识

| 标志名称      | 标志值 | 含义                       |
| ------------- | ------ | -------------------------- |
| ACC_PUBLIC    | 0x0001 | 字段是否为public           |
| ACC_PRIVATE   | 0x0002 | 字段是否为private          |
| ACC_PROTECTED | 0x0004 | 字段是否为protected        |
| ACC_STATIC    | 0x0008 | 字段是否为static           |
| ACC_FINAL     | 0x0010 | 字段是否为final            |
| ACC_VOLATILE  | 0x0040 | 字段是否为volatile         |
| ACC_TRANSTENT | 0x0080 | 字段是否为transient        |
| ACC_SYNCHETIC | 0x1000 | 字段是否为由编译器自动产生 |
| ACC_ENUM      | 0x4000 | 字段是否为enum             |

#### 2.7.2.2.字段名索引

根据字段名索引的值,查询常量池中的指定索引;

#### 2.7.2.3.描述符索引

描述符的作用是用来描述字段的数据类型,方法的参数列表和返回值.

| 标志符 | 含义                                                 |
| ------ | ---------------------------------------------------- |
| B      | 基本数据类型byte                                     |
| C      | 基本数据类型char                                     |
| D      | 基本数据类型double                                   |
| F      | 基本数据类型float                                    |
| I      | 基本数据类型int                                      |
| J      | 基本数据类型long                                     |
| S      | 基本数据类型short                                    |
| Z      | 基本数据类型boolean                                  |
| V      | 代表void类型                                         |
| L      | 对象类型，比如：`Ljava/lang/Object;`                 |
| [      | 数组类型，代表一维数组。比如：`double[][][] is [[[D` |

#### 2.7.2.4.属性表集合

一个字段还可能拥有一些属性,用于存储更多的额外的信息,比如初始化值,一些注释信息,属性个数存放在`attribute_count`,属性具体内容存放在`attributes[]`中;

以常量属性为例,结构为:

* `ConstantValue_attribute`;
  * `u2 attribute_name_index`;
  * `u4 attribute_length`;
  * `u2 constantvalue_index`;

## 2.8.方法表集合

methods:指向常量池索引集合,完整描述了每个方法的签名;

* 在字节码文件中,每一个method_info都对应着一个类或者接口中的方法信息.比如方法的访问修饰符(public,private或protected),方法的返回值类型以及方法的参数信息等;
* 如果这个方法不是抽象或者native的,那么字节码中会体现出来;
* methods表只描述当前类或接口中声明的方法,不包括从父类或者父接口中继承的方法;
* methods表有可能出现有编译器自动添加的方法,如类(接口)初始化方法`<clinit>()`和实例初始化方法`<init>()`;

### 2.8.1.方法计数器(略)

### 2.8.2.方法表

方法表的结构跟字段表是一样的,结构如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027212014682.png" alt="image-20211027212014682" style="zoom:67%;" />

#### 2.8.2.1.方法表访问标识(略)

#### 2.8.2.2.方法名索引(略)

#### 2.8.2.3.描述符索引(略)

#### 2.8.2.4.属性表集合(见2.9)

## 2.9.属性表集合

* 方法表之后的属性表集合,指的是class文件所携带的辅助信息,比如该class文件的源文件名称;
* 字段表,方法表都可以有自己的属性表,用于描述某些场景专有的信息;

| 类型 | 名称                 | 数量             | 含义       |
| ---- | -------------------- | ---------------- | ---------- |
| u2   | attribute_name_index | 1                | 属性名索引 |
| u4   | attribute_length     | 1                | 属性长度   |
| u1   | info                 | attribute_length | 属性表     |

属性表集合比较复杂,可以参考官网https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7

![image-20211027224525864](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211027224525864.png)

# 3.`Javap`指令

~~~shell
➜  ~ javap -help
用法: javap <options> <classes>
其中, 可能的选项包括:
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
~~~

最全的显示方式`javap -v -p xxxxx.class`;

javap字节码文件解析如下

~~~shell
Classfile /C:/Users/songhk/Desktop/2/JavapTest.class    //字节码文件所属的路径
  Last modified 2020-9-7; size 1358 bytes		//最后修改时间，字节码文件的大小
  MD5 checksum 526b4a845e4d98180438e4c5781b7e88         //MD5散列值
  Compiled from "JavapTest.java"			//源文件的名称
public class com.atguigu.java1.JavapTest
  minor version: 0					//副版本
  major version: 52					//主版本
  flags: ACC_PUBLIC, ACC_SUPER				//访问标识
Constant pool:						//常量池
   #1 = Methodref          #16.#46        // java/lang/Object."<init>":()V
   #2 = String             #47            // java
   #3 = Fieldref           #15.#48        // com/atguigu/java1/JavapTest.info:Ljava/lang/String;
   #4 = Fieldref           #15.#49        // com/atguigu/java1/JavapTest.flag:Z
   #5 = Fieldref           #15.#50        // com/atguigu/java1/JavapTest.num:I
   #6 = Fieldref           #15.#51        // com/atguigu/java1/JavapTest.gender:C
   #7 = Fieldref           #52.#53        // java/lang/System.out:Ljava/io/PrintStream;
   #8 = Class              #54            // java/lang/StringBuilder
   #9 = Methodref          #8.#46         // java/lang/StringBuilder."<init>":()V
  #10 = Methodref          #8.#55         // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #11 = Methodref          #8.#56         // java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
  #12 = Methodref          #8.#57         // java/lang/StringBuilder.toString:()Ljava/lang/String;
  #13 = Methodref          #58.#59        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #14 = String             #60            // www.atguigu.com
  #15 = Class              #61            // com/atguigu/java1/JavapTest
  #16 = Class              #62            // java/lang/Object
  #17 = Utf8               num
  #18 = Utf8               I
  #19 = Utf8               flag
  #20 = Utf8               Z
  #21 = Utf8               gender
  #22 = Utf8               C
  #23 = Utf8               info
  #24 = Utf8               Ljava/lang/String;
  #25 = Utf8               COUNTS
  #26 = Utf8               ConstantValue
  #27 = Integer            1
  #28 = Utf8               <init>
  #29 = Utf8               ()V
  #30 = Utf8               Code
  #31 = Utf8               LineNumberTable
  #32 = Utf8               LocalVariableTable
  #33 = Utf8               this
  #34 = Utf8               Lcom/atguigu/java1/JavapTest;
  #35 = Utf8               (Z)V
  #36 = Utf8               methodPrivate
  #37 = Utf8               getNum
  #38 = Utf8               (I)I
  #39 = Utf8               i
  #40 = Utf8               showGender
  #41 = Utf8               ()C
  #42 = Utf8               showInfo
  #43 = Utf8               <clinit>
  #44 = Utf8               SourceFile
  #45 = Utf8               JavapTest.java
  #46 = NameAndType        #28:#29        // "<init>":()V
  #47 = Utf8               java
  #48 = NameAndType        #23:#24        // info:Ljava/lang/String;
  #49 = NameAndType        #19:#20        // flag:Z
  #50 = NameAndType        #17:#18        // num:I
  #51 = NameAndType        #21:#22        // gender:C
  #52 = Class              #63            // java/lang/System
  #53 = NameAndType        #64:#65        // out:Ljava/io/PrintStream;
  #54 = Utf8               java/lang/StringBuilder
  #55 = NameAndType        #66:#67        // append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #56 = NameAndType        #66:#68        // append:(I)Ljava/lang/StringBuilder;
  #57 = NameAndType        #69:#70        // toString:()Ljava/lang/String;
  #58 = Class              #71            // java/io/PrintStream
  #59 = NameAndType        #72:#73        // println:(Ljava/lang/String;)V
  #60 = Utf8               www.atguigu.com
  #61 = Utf8               com/atguigu/java1/JavapTest
  #62 = Utf8               java/lang/Object
  #63 = Utf8               java/lang/System
  #64 = Utf8               out
  #65 = Utf8               Ljava/io/PrintStream;
  #66 = Utf8               append
  #67 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #68 = Utf8               (I)Ljava/lang/StringBuilder;
  #69 = Utf8               toString
  #70 = Utf8               ()Ljava/lang/String;
  #71 = Utf8               java/io/PrintStream
  #72 = Utf8               println
  #73 = Utf8               (Ljava/lang/String;)V
#######################################字段表集合的信息################################################
{						
  private int num;				//字段名
    descriptor: I				//字段描述符：字段的类型
    flags: ACC_PRIVATE				//字段的访问标识

  boolean flag;
    descriptor: Z
    flags:

  protected char gender;
    descriptor: C
    flags: ACC_PROTECTED

  public java.lang.String info;
    descriptor: Ljava/lang/String;
    flags: ACC_PUBLIC

  public static final int COUNTS;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 1				//常量字段的属性：ConstantValue

#######################################方法表集合的信息################################################
  public com.atguigu.java1.JavapTest();				//构造器1的信息		
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String java
         7: putfield      #3                  // Field info:Ljava/lang/String;
        10: return
      LineNumberTable:
        line 20: 0
        line 18: 4
        line 22: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcom/atguigu/java1/JavapTest;

  private com.atguigu.java1.JavapTest(boolean);			//构造器2的信息	
    descriptor: (Z)V
    flags: ACC_PRIVATE
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String java
         7: putfield      #3                  // Field info:Ljava/lang/String;
        10: aload_0
        11: iload_1
        12: putfield      #4                  // Field flag:Z
        15: return
      LineNumberTable:
        line 23: 0
        line 18: 4
        line 24: 10
        line 25: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  this   Lcom/atguigu/java1/JavapTest;
            0      16     1  flag   Z

  private void methodPrivate();
    descriptor: ()V
    flags: ACC_PRIVATE
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 28: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcom/atguigu/java1/JavapTest;

  int getNum(int);
    descriptor: (I)I
    flags:
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: getfield      #5                  // Field num:I
         4: iload_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 30: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/atguigu/java1/JavapTest;
            0       7     1     i   I

  protected char showGender();
    descriptor: ()C
    flags: ACC_PROTECTED
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #6                  // Field gender:C
         4: ireturn
      LineNumberTable:
        line 33: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/atguigu/java1/JavapTest;

  public void showInfo();				
    descriptor: ()V				//方法描述符：方法的形参列表 、 返回值类型
    flags: ACC_PUBLIC				//方法的访问标识
    Code:					//方法的Code属性
      stack=3, locals=2, args_size=1		//stack:操作数栈的最大深度   locals:局部变量表的长度  args_size：方法接收参数的个数
   //偏移量 操作码	 操作数	
	 0: bipush        10
         2: istore_1
         3: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
         6: new           #8                  // class java/lang/StringBuilder
         9: dup
        10: invokespecial #9                  // Method java/lang/StringBuilder."<init>":()V
        13: aload_0
        14: getfield      #3                  // Field info:Ljava/lang/String;
        17: invokevirtual #10                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        20: iload_1
        21: invokevirtual #11                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        24: invokevirtual #12                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        27: invokevirtual #13                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        30: return
      //行号表：指名字节码指令的偏移量与java源程序中代码的行号的一一对应关系
      LineNumberTable:
        line 36: 0
        line 37: 3
        line 38: 30
      //局部变量表：描述内部局部变量的相关信息
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      31     0  this   Lcom/atguigu/java1/JavapTest;
            3      28     1     i   I

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=1, args_size=0
         0: ldc           #14                 // String www.atguigu.com
         2: astore_0
         3: return
      LineNumberTable:
        line 15: 0
        line 16: 3
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
}
SourceFile: "JavapTest.java"			//附加属性：指名当前字节码文件对应的源程序文件名

~~~

