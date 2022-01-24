#    JVM详解02

# 1.运行时数据区

![image-20210801133751805](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801133751805.png)

JDK1.8之后的运行时数据区如下:

![image-20210801134605296](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801134605296.png)

* 线程独立:包括程序计数器,栈,本地栈;
* 线程间共享:堆,堆外内存(永久代或元空间,代码缓存);

![image-20210819100949730](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819100949730.png)

## 1.1.线程

### 1.1.2.JVM线程与操作系统线程

* 在HotSpot VM里,每个线程都与操作系统的本地线程直接映射;当一个Java线程准备好执行后,此时一个操作系统的本地线程也会同时创建.Java线程执行终止后,本地线程也会回收;
* 操作系统负责所有线程的安排调度到任何一个可用的CPU上.一旦本地线程初始化成功,它就会调用Java线程中的`run()`方法;

### 1.1.3.JVM系统线程

在HotSpot JVM中主要是如下的后台系统线程:

* 虚拟机线程:这种线程的操作是需要JVM达到安全点才会出现,这些操作必须在不同的线程中发生的原因是他们都需要JVM的安全点,这样堆才不会变化,这种线程的执行包括"stop-the-world"的垃圾收集,线程栈收集,线程挂起以及偏向锁的撤销;
* 周期任务线程:这种线程是时间周期事件的体现(比如中断),他们一般用于周期性的操作调度执行;
* GC线程:这种线程对JVM立面不同种类的垃圾收集行为提供了支持;
* 编译线程:这种线程在运行时会将字节码编译到本地代码;
* 信号调度线程:这种线程接收信号并发送给JVM,在它内部通过调用适当的方法进行处理;

## 1.2.程序计数器(PC寄存器)

JVM中的PC寄存器是对物理PC寄存器的一种抽象模拟;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801143521061.png" alt="image-20210801143521061" style="zoom: 33%;" />

PC寄存器用来存储指向下一条指令的地址,也即将要执行的指令代码.由执行引擎读取下一条指令

* 它是程序控制流的指示器,分支,循环,跳转,异常处理,线程恢复等基础功能都需要依赖这个计数器来完成;
* 字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令;
* 它是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域;

~~~java
public class PcRegisterTest {

    public static void main(String[] args) {
        int i = 3;
        int j = 4;
        int k = i + j;
    }
}
~~~

通过`javap`如下图

![image-20210801160206234](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801160206234.png)

红色标记是偏移地址,指令地址,PC寄存器存储的就是这个; 

### 1.2.1.为什么使用PC寄存器记录当前线程的执行地址?

因为CPU需要不停的切换各个线程,这个时候切换回来的时候,就得知接着从哪开始继续执行;

## 1.3.栈

* 每个线程在创建时都会创建一个虚拟机栈,其内部保存一个个的栈帧,对应着一次次的Java方法调用,是线程私有的;
* 生命周期和线程一致;
* 主管Java程序的运行,它保存方法的局部变量(8种基本数据类型,对象的引用地址),部分结果,并参与方法的调用与返回;

### 1.3.1.虚拟机栈的异常

* Java虚拟机规范允许Java栈的大小是**动态或者固定不变的**;
  * 如果采用固定大小的Java虚拟机栈,那每一个线程的Java虚拟机栈容量可以在线程创建的时候独立选定.如果线程请求分配的栈容量大小超过了Java虚拟机栈允许的最大容量,Java虚拟机将会抛出`StackOverflowError`;
  * 如果Java虚拟机栈可以动态扩展,并且在尝试扩展的时候无法申请到足够的内存,或者在创建新的线程时候没有足够内存去创建对应的虚拟机栈,那么Java虚拟机会抛出`OutOfMemoeryError`异常;

### 1.3.2.设置栈大小

查看官方文档:

https://docs.oracle.com/javase/8/

https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html#BGBCIEFC

或者在Oracle上直接将Java Document下载到本地观看

[Java Document]: file:///Users/fechinchu/Develop/java-docs/docs/technotes/tools/unix/java.html

![image-20210801165810996](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801165810996.png)

### 1.3.3.栈的运行原理

* 在一条活动线程中,一个时间点上,只会有一个活动的栈帧.即只有当前正在执行的方法的栈帧(栈顶栈帧)是有效的,这个栈帧被称为当前栈帧,与当前栈帧相对应的方法就是当前方法,定义这个方法的类就是当前类;
* 执行引擎运行的所有字节码指令只针对当前栈帧进行操作;
* 不同线程中所包含的栈帧是不允许存在相互引用的,即不可能在一个栈帧中引用另外一个线程的栈帧;

### 1.3.4.栈帧的内部结构

* LV:local variable:局部变量表;
* OS:operand stack:操作数栈(表达式栈);
* DL:dynamic link:动态链接(指向运行时常量池的方法引用);
* RA:return address:方法返回地址(方法正常退出或异常退出的定义);
* 一些附加信息;

#### 1.3.4.1.局部变量表

* 局部变量表定义为一个数字数组,主要用于存储方法参数和定义在方法体内部的局部变量,这些数据类型包括各类基本数据类型,对象引用(reference),以及returnAddress类型;
* 由于局部变量表是建立在线程的栈上,是线程的私有数据,因此不存在数据安全问题;
* 局部变量表所需的容量是在编译期确定下来的,并保存在方法的Code属性的maximum lcoal variables数据项中.在方法运行期间是不会改变局部变量表的大小的;  
* 参数值的存放总是在局部变量数组的index0开始,到数组长度-1的索引结束;
* 局部变量表,最基本的存储是slot(变量槽);
* 局部变量表中存放编译器可知的各种基本数据类型(8种),引用类型(reference),returnAddress类型;
* 在局部变量表中,32位以内的类型只占用一个slot(包括returnAddress类型),64位的类型(long和double)占用两个slot;
  * byte,short,char在存储前被转化为int,boolean也被转化为int,0表示false,非0表示true;
  * long和double占据两个slot;
* 方法嵌套调用的次数由栈的大小决定.
* 局部变量表中的变量只在当前方法调用中有效.在方法执行时,虚拟机通过使用局部变量完成参数值到参数变量列表的传递过程.当方法调用结束后,随着方法栈帧的销毁,局部变量表也会随之销毁;
* 当一个实例方法被调用的时候,它的方法参数和方法体内部定义的局部变量将会按照顺序被复制到局部变量表中的每一个Slot上;
* 如果当前帧是由构造方法或者实例方法创建的,那么该对象引用this将会存放在index为0的slot处,其余的参数按照参数表顺序排列;

```java
public class LocalVariablesTest {
    private int count = 0;

    public static void main(String[] args) {
        LocalVariablesTest test = new LocalVariablesTest();
        int num = 10;
        test.test1();
    }

    public void test1() {
        Date date = new Date();
        String name1 = "fechin";
        test2(date, name1);
        System.out.println(date + name1);
    }

    public String test2(Date dateP, String name2) {
        dateP = null;
        name2 = "zhuguoqing";
        double weight = 130.5;//占据两个slot
        char gender = '男';
        return dateP + name2;
    }
}
```

![image-20210802195333135](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802195333135.png)

上图标注的就是局部变量表

> 通过上述代码的例子来说明字节码的相关数据
>
> ![image-20210802205946856](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802205946856.png)这里描述符:`<(Ljava/util/Date;Ljava/lang/String;)Ljava/lang/String;>`表示的是两个参数一个是Date类型,一个是String类型,返回值是String类型,`L`表示的是引用类型;访问标识是public
>
> ![image-20210802210112792](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802210112792.png)
>
> 上图是Misc信息;字节码长度33,可以通过字节码观察到如下图,长度为33
>
> ![image-20210802210330063](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802210330063.png)
>
> ![image-20210802210516753](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802210516753.png)
>
> Start PC(起始PC)是字节码的行号,Line Number(行号)是Java代码的行号;
>
> ![image-20210802210815250](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802210815250.png)
>
> Start PC(起始PC)表示的是字节码指令的行号是该变量作用域的起始位置,Length(长度)表示的是作用域的长度;

如下是局部变量表:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210802220551973.png" alt="image-20210802220551973" style="zoom:50%;" />

> * 为什么静态方法中不能使用`this`?
>   * 因为this变量不存在与当前方法的局部变量表中

* 栈帧中的局部变量表中的槽位是可以重用的,如果一个局部变量过了其作用域,那么在其作用域之后申明的新的局部变量就很有可能会复用过期局部变量的槽位.从而达到节省资源的目的
* 和类变量初始化不同的是,局部变量表不存在系统初始化的过程,这意味着一旦定义了局部变量则必须人为的初始化,否则无法使用;
* 局部变量表中的变量是重要的垃圾回收根节点,只要被局部变量表中直接或者间接引用的对象都不会被回收引用;

#### 1.3.4.2.操作数栈

操作数栈,在方法执行过程中,根据字节码指令,往栈中写入数据或提取数据,即入栈(push)/出栈(pop);

某些字节码指令将值压入操作数栈,其余的字节码指令将操作数取出栈,使用后再把结果压入栈.下图就是操作数栈

![image-20210805150537943](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210805150537943.png)

```java
public static void main(String[] args) {
    int i = 2;
    int j = 3;
    int k = i + j;
}
```

* 操作数栈,主要用于保存计算过程的中间结果,同时作为计算过程中变量的临时的存储空间;
* 操作数栈就是JVM执行引擎的一个工作区,当一个方法刚开始执行的时候,一个新的栈帧也会随之被创建出来,这个方法的操作数栈是空的;
* 每一个操作数栈都会拥有一个明确的栈深度用于存储数值,其所需的最大深度在编译期就定好了,保存在方法的Code属性中,为max_stack的值;
* 栈中的任何一个元素都可以任意的Java数据类型;
  * 32Bit的类型占用一个栈单位深度;
  * 64Bit的类型占用两个站单位深度;
* 操作数栈并非采用访问索引的方式来进行数据访问,虽然它用的是数组这个数据结构,而是只能通过标准的入栈和出栈操作来完成一次数据访问;
* 如果被调用的方法带有返回值的话,其返回值将会被压入当前栈的操作数栈中,并更新PC寄存器中下一条需要执行的字节码指令;
* 操作数栈中元素的数据类型必须与字节码指令的序列严格匹配,这有编译器在编译期间进行验证,同时在类加载过程中的类检验阶段的数据分析阶段再次验证;
* Java虚拟机的解释引擎是基于栈的执行引擎,这里的栈指的就是操作数栈;

 如下是针对操作数栈的代码追踪:

~~~java
public void testAddOperation() {
        //byte、short、char、boolean：都以int型来保存
        byte i = 15;
        int j = 8;
        int k = i + j;
}
~~~

![image-20210805155622909](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210805155622909.png)

![image-20210805160702964](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210805160702964.png)

基于栈式架构的虚拟机所使用的零地址指令更加紧凑,但完成一项操作的时候必须需要使用更多的入栈和出栈指令,这同时也就意味着将需要更多的指令分派次数和内存读/写次数;由于操作数是存储在内存中的,因此频繁地执行内存读/写操作必然会影响执行速度.为了解决这个问题,HotSpot JVM设计了**栈顶缓存技术**,栈顶缓存技术将栈顶元素全部缓存在物理CPU的寄存器中,以此降低对内存的读/写次数,提升执行引擎的效率;

#### 1.3.4.3.动态链接

![image-20210817151901935](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817151901935.png)

* 每一个栈帧内部都包含一个指向运行时常量池中该栈帧所属方法的引用.包含这个引用的目的就是为了支持当前方法的代码能够实现动态链接.比如:`invokedynamic`指令;
* 在Java源文件中被编译到字节码文件中,所有的变量和方法引用都作为符号引用保存在class文件的常量池中.比如:描述一个方法调用了另外的其他的方法时,就是通过常量池中指向方法的符号引用来表示的,那么**动态链接的作用就是为了将这些符号引用转换为调用方法的直接引用**;

~~~java
public class DynamicLinkingTest {

    int num = 10;

    public void methodA(){
        System.out.println("methodA()....");
    }

    public void methodB(){
        System.out.println("methodB()....");
        methodA();
        num++;
    }
}
~~~

![image-20210817150648327](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817150648327.png)

如上图是`methodB()`的字节码文件.上图中圈起来的就是调用常量池中指向方法的符号引用.

![image-20210817150906112](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817150906112.png)

> 在JVM中,将符号引用转换为调用方法的直接引用与方法的绑定机制有关;
>
> * **静态链接**:当一个字节码文件被装载进JVM内部时,如果被调用的目标方法在**编译期**可知,且运行期保持不变.这种情况下将调用方法的**符号引用**转换为**直接引用**的过程称为静态链接;
>
> * **动态链接**:如果被调用的方法在编译期无法被确定下来,也就是说,只能够在程序**运行期**将调用方法的**符号引用**转换为**直接引用**,由于这种引用转换过程具备动态性,因此被称为动态链接;
>
> 对应的方法的绑定机制是**早期绑定**和**晚期绑定**;
>
> 
>
>  Java中的多态也正是因为动态链接和晚期绑定才得以实现;如果在Java程序中不希望某个方法拥有虚函数(晚期绑定)的特征时,可以使用关键字final来标记;
>
> * **非虚方法**:如果方法在编译期就确定了具体的调用版本,这个版本在运行时是不可变的,这样的方法称为非虚方法;
>   * 静态方法,私有方法,final方法,实例构造器,父类方法都是非虚方法;
> * **虚方法**:其余方法被称为虚方法;
>
> 
>
> 虚拟机中提供了以下方法调用指令:
>
> * 普通调用指令:
>   * invokestatic:调用静态方法,解析阶段确定唯一方法版本;(非虚方法)
>   * invokespecial:调用`<init>`方法,是由及父类方法,解析阶段确定唯一方法版本;(非虚方法)
>   * invokevirtual:调用所有虚方法;(除final修饰,都是虚方法)
>   * invokeinterface:调用接口方法;(除final修饰,都是虚方法)
> * 动态调用指令:
>   * invokedynamic:动态解析出需要调用的方法,然后执行;
>
> 
>
> 动态类型语言和静态类型语言:
>
> 两者的区别就是在于对类型的检查是在编译器还是运行期,满足前者就是静态类型语言,满足后者就是动态类型语言;
>
> * **静态类型**语言是判断变量自身的类型信息;
> * **动态类型**语言是判断变量值的类型信息,变量没有信息,变量值才有类型信息;
>
> 
>
> Java语言重写的本质:
>
> 1. 找到操作数栈顶的第一个元素所执行的对象的实际类型,记做C;
> 2. 如果在类型C中找到与常量中的描述符合简单名称都相符合的方法,则进行访问权限的校验;如果通过返回这个方法的直接引用,查找过程结束;如果不通过,则返回`java.lang.IllegalAccessError`异常;
> 3. 否则,按照继承关系从下往上依次对C的各个父类进行第2步搜索和验证过程;
> 4. 如果始终没有找到合适的方法,则抛出`java.lang.AbstractMethodError`异常;
>
> 在面向对象的编程中,会很频繁的使用到动态分派,如果在每次动态分派的过程中都要重新在类的方法元数据中搜索合适的目标就可能影响到执行效率.因此,为了提高性能,JVM采用在类的方法区建立一个虚方法(不包含非虚方法)表来实现.它会在类加载的链接阶段被创建并初始化;类的变量初始值准备完成之后,JVM会把该类的方法表也初始化完毕;



~~~java
class Animal{
    public void eat(){
        System.out.println("动物进食");
    }
}

interface Huntable{
    void hunt();
}

class Dog extends Animal implements Huntable{
    @Override
    public void eat() {
        System.out.println("狗吃骨头");
    }

    @Override
    public void hunt() {
        System.out.println("捕食耗子，多管闲事");
    }
}

class Cat extends Animal implements Huntable{

    public Cat(){
        super();//表现为：早期绑定 invokespecial
    }

    public Cat(String name){
        this();//表现为：早期绑定 invokespecial
    }

    @Override
    public void eat() {
        super.eat();//表现为：早期绑定
        System.out.println("猫吃鱼");
    }

    @Override
    public void hunt() {
        System.out.println("捕食耗子，天经地义");
    }
}
public class AnimalTest {
    public void showAnimal(Animal animal){
        animal.eat();//表现为：晚期绑定 invokevirtual
    }
    public void showHunt(Huntable h){
        h.hunt();//表现为：晚期绑定 invokeinterface
    }
}
~~~

~~~java
class Father {
    public Father() {
        System.out.println("father的构造器");
    }

    public static void showStatic(String str) {
        System.out.println("father " + str);
    }

    public final void showFinal() {
        System.out.println("father show final");
    }

    public void showCommon() {
        System.out.println("father 普通方法");
    }
}

public class Son extends Father {
    public Son() {
        //invokespecial
        super();
    }
    public Son(int age) {
        //invokespecial
        this();
    }
    //不是重写的父类的静态方法，因为静态方法不能被重写！
    public static void showStatic(String str) {
        System.out.println("son " + str);
    }
    private void showPrivate(String str) {
        System.out.println("son private" + str);
    }

    public void show() {
        //invokestatic
        showStatic("hi");
        //invokestatic
        super.showStatic("good!");
        //invokespecial
        showPrivate("hello!");
        //invokespecial
        super.showCommon();

        //invokevirtual
        showFinal();//因为此方法声明有final，不能被子类重写，所以也认为此方法是非虚方法。
        //虚方法如下：
        //invokevirtual
        showCommon();
      	//invokevirtual
        info();

        MethodInterface in = null;
        //invokeinterface
        in.methodA();
    }

    public void info(){

    }

    public void display(Father f){
        f.showCommon();
    }

    public static void main(String[] args) {
        Son so = new Son();
        so.show();
    }
}

interface MethodInterface{
    void methodA();
}
~~~

~~~java
@FunctionalInterface
interface Func {
    public boolean func(String str);
}

public class Lambda {
    public void lambda(Func func) {
        return;
    }

    public static void main(String[] args) {
        Lambda lambda = new Lambda();
        //invokedynamic
        Func func = s -> {
            return true;
        };

        lambda.lambda(func);
        //invokedynamic
        lambda.lambda(s -> {
            return true;
        });
    }
}
~~~

#### 1.3.4.4.方法返回地址

方法返回地址存放的是**调用该方法的PC寄存器的值**;

不管是正常退出还是异常退出,在方法退出后都返回到该方法被调用的位置.方法正常退出时,调用者的PC计数器的值作为返回地址,即调用该方法的指令的下一条指令的地址.而通过异常退出的,返回地址是要通过异常表来确定,栈帧中一般不会保存这部分信息;

异常处理表如下:

~~~java
public class ReturnAddressTest {
    public boolean methodBoolean() {
        return false;
    }

    public byte methodByte() {
        return 0;
    }

    public short methodShort() {
        return 0;
    }

    public char methodChar() {
        return 'a';
    }

    public int methodInt() {
        return 0;
    }

    public long methodLong() {
        return 0L;
    }

    public float methodFloat() {
        return 0.0f;
    }

    public double methodDouble() {
        return 0.0;
    }

    public String methodString() {
        return null;
    }

    public Date methodDate() {
        return null;
    }

    public void methodVoid() {

    }

    static {
        int i = 10;
    }

    public void method2() {
        methodVoid();
        try {
            method1();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void method1() throws IOException {
        FileReader fis = new FileReader("fechin.txt");
        char[] cBuffer = new char[1024];
        int len;
        while ((len = fis.read(cBuffer)) != -1) {
            String str = new String(cBuffer, 0, len);
            System.out.println(str);
        }
        fis.close();
    }
}
~~~

![image-20210817171732872](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817171732872.png)

#### 1.3.4.5.一些附加信息

栈帧中允许携带与Java虚拟机实现相关的一些附加信息.列如:对程序调试提供支持的信息;

 ### 1.3.5.栈中的局部变量是否线程安全

~~~java
public class StringBuilderTest {

    int num = 10;

    //s1的声明方式是线程安全的
    public static void method1(){
        //StringBuilder:线程不安全
        StringBuilder s1 = new StringBuilder();
        s1.append("a");
        s1.append("b");
        //...
    }
    //sBuilder的操作过程：是线程不安全的
    public static void method2(StringBuilder sBuilder){
        sBuilder.append("a");
        sBuilder.append("b");
        //...
    }
    //s1的操作：是线程不安全的
    public static StringBuilder method3(){
        StringBuilder s1 = new StringBuilder();
        s1.append("a");
        s1.append("b");
        return s1;
    }
    //s1的操作：是线程安全的
    public static String method4(){
        StringBuilder s1 = new StringBuilder();
        s1.append("a");
        s1.append("b");
        return s1.toString();
    }

    public static void main(String[] args) {
        StringBuilder s = new StringBuilder();
        new Thread(() -> {
            s.append("a");
            s.append("b");
        }).start();
        method2(s);
    }

}
~~~

具体问题具体分析,如果发生的逃逸,则线程不安全;

## 1.4.堆

编写代码加入参数`-Xms10m -Xmx10m`,运行后使用`jvisualvm`在控制台调出Java VisualVM,下图圈出来的就是堆

![image-20210817203200161](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817203200161.png)

* 堆是JVM管理的最大的一块内存空间;
* 堆可以处于物理上不连续的内存空间,但是在逻辑上应该被视为连续;
* 所有的线程共享Java堆,在这里还可以划分线程私有的缓冲区(Thread Local Allocation Buffer,TLAB);

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817212028296.png" alt="image-20210817212028296" style="zoom: 67%;" />

### 1.5.1.堆的内存细分

* Java 7及之前堆的内存分为三部分:
  * 新生代(Young Generation Space):Young/New
    * Eden区;
    * Survivor区;
  * 老年代(Tenure Generation Sapce) :Old/Tenure;
  * 永久区:Perm
* Java 8及之后堆内存分为三部分:
  * 新生代(Young Generation Space):Young/New
    * Eden区;
    * Survivor区;
  * 老年代(Tenure Generation Sapce):Old/Tenure;
  * 元空间(Meta Space):Meta;

![image-20210817213733551](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817213733551.png)

### 1.5.2.堆空间大小的设置

Java堆区用于存储Java对象实例,那么堆大小在JVM启动时就设定好了:

* `-Xms`用于表示堆的起始内存:等价于`-XX:InitialHeapSize`;
* `-Xmx`:用于表示堆的最大内存:等价于`-XX:MaxHeapSize`;
* 堆超过`-Xmx`指定的最大内存,将会抛出OOM;
* 通常会将`-Xms`和`-Xmx`两个参数配置相同的值,目的是**为了能够在Java垃圾回收机制清理完堆后不需要重新份额计算堆区大小,提高性能**;
* 默认情况下:
  * 初始内存大小:物理内存大小/64;
  * 最大内存大小:物理内存大小/4;
* 查看设置的参数:
  * 方式一:`jstat -gc [进程id]`;
  * 方式二:`-XX:+PrintGCDetails`;

* 配置新生代与老年代在堆结构的占比(开发中一般不会调整);默认`-XX:NewRatio=2`,表示新生代占1,老年代占2,新生代占整个堆的1/3;
* Eden空间和另外两个Survivor空间默认是8:1:1,实际的默认并不是8:1:1.官方描述与实际并不符;可以通过`-XX:SurvivorRatio`来调整这个空间比例.比如:`-XX:SurvivorRatio=8`;
* 可以使用`-Xmn`设置新生代最大内存大小;一般默认即可;
* `-XX:-UseAdaptiveSizePolicy`:关闭自适应的内存分配策略;

### 1.5.3.对象分配过程

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817231217440.png" alt="image-20210817231217440" style="zoom: 50%;" />

* new的对象先放Eden,当Eden区满后,JVM会对Eden区进行垃圾回收(Minor GC).将Eden区不再被其他对象引用的对象进行销毁.然后将Eden区剩余对象放到Survivor区;
* Survivor区去Old区需要一个阈值,默认是15,可以设置参数`-XX:MaxTenuringThreshold=[N]`进行设置;
* 动态对象年龄判断:如果Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半,年龄大于或等于该年龄的对象可以直接进入老年代,无需等待MaxTrnuringThreshold中要求的年龄;
* 空间分配担保:`-XX:HandlePromotionFailure`;
* 当老年代内存不足时候,再次触发GC:Major GC进行老年代的内存清理,如果老年代执行Major GC后发现依然无法进行对象的保存,就会OOM; 

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210817233427527.png" alt="image-20210817233427527" style="zoom: 67%;" />

  ### 1.5.4.Minor GC,Major GC,Full GC

针对HotSpot VM,GC按照回收区域分为两大种类型:一种是部分收集(Partial GC),一种是整堆收集(Full GC);

* 部分收集(Partial GC):
  * 新生代收集(Minor GC/Young GC):只是新生代的垃圾收集;
  * 老年代收集(Major GC/Old GC):只是老年代的垃圾收集;
    * 只有CMS GC会有单独收集老年代的行为;
  * 混合收集(Mixed GC):收集整个新生代以及部分老年代的垃圾收集;
    * 只有G1 GC会有这种行为;
* 整堆收集(Full GC):收集整个Java堆和方法区的垃圾收集;

> * Minor GC机制:
>   * 当年轻代空间不足时候,就会触发Minor GC.这里的年轻代满是Eden满,Survivor满不会触发GC,Minor GC清理年轻代的内存;
>   * Minor GC非常频繁,一般回收速度也很快;
>   * Minor GC会触发STW,暂停其他用户线程;
> * Major GC机制:
>   * 出现Major GC,经常会伴随至少一次Minor GC(并非绝对,Parallel Scavenge收集器的收集策略中有直接进行Major GC的策略);
>   * Major GC一般会比Minor GC满10倍以上,STW时间更长;
>   * 如果Major GC以后,内存还是不足,就报OOM;
> * Full GC机制:
>   * `System.gc()`,系统建议执行Full GC,但是不必然执行;
>   * 老年代空间不足;
>   * 方法区空间不足;
>   * 通过Minor GC后进入老年代的平均大小大于老年代的可用内存;

### 1.5.5.Thread Local Allocation Buffer

* 从内存模型而不是垃圾收集的角度,对Eden区域继续进行划分,JVM为每个线程分配一个私有缓存区域,包含在Eden空间内;
* 多线程同时分配内存时,使用TLAB可以避免一系列的非线程安全问题.同时还能够提升内存分配的吞吐量,将这种内存分配方式叫做快速分配策略;
* 不是所有的对象都能够在TLAB中成功分配内存,但JVM确实将TLAB作为内存分配的首选;
* 可以通过`-XX:UseTLAB`设置是否开启TLAB空间;
* TLAB默认栈整个Eden空间的1%,可以通过`-XX:TLABWasteTargetPercent`设置TLAB空间占用Eden空间的百分比大小;
* 一旦对象在TLAB空间分配内存失败时,JVM就会尝试使用加锁机制确保数据操作的原子性,从而直接在Eden空间中分配内存;
* 可以通过`jinfo -flag UseTLAB [PID]`来看是否开启了TLAB;

如下是加上TLAB的对象的分配过程:

![image-20210818143200784](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818143200784.png)

### 1.5.6.堆空间的参数

 ~~~shell
 -XX:+PrintFlagsInitial #查看所有的参数的默认初始值
 -XX:+PrintFlagsFinal #查看所有的参数的最终值（可能会存在修改，不再是初始值）
 jinfo -flag [SurvivorRatio] [进程id] #查看参数设置的值
 -Xms：#初始堆空间内存 （默认为物理内存的1/64）
 -Xmx：#最大堆空间内存（默认为物理内存的1/4）
 -Xmn：#设置新生代的大小。(初始值及最大值)
 -XX:NewRatio：#配置新生代与老年代在堆结构的占比
 -XX:SurvivorRatio：#设置新生代中Eden和S0/S1空间的比例
 -XX:MaxTenuringThreshold：#设置新生代垃圾的最大年龄
 -XX:+PrintGCDetails #输出详细的GC处理日志
 -XX:+PrintGC #打印GC概要
 -verbose:gc #打印GC概要
 -XX:HandlePromotionFailure：#是否设置空间分配担保,JDK6 update24之后,该参数不会再影响到虚拟机空间分配担保策略.规则变成了只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC,否则会进行Full GC;
 ~~~

   ### 1.5.7.逃逸分析

使用逃逸分析`-XX:+DoEscapeAnalysis`,默认是开启的,编译器可以对代码做如下优化

#### 1.5.7.1.栈上分配 

JIT编译器在编译期间根据逃逸分析的结果,发现如果一个对象没有逃逸出方法的话,就可能被优化成栈上分配.分配完成后,继续在调用栈内执行;最后线程结束,栈空间被回收,局部变量对象也会被回收.无需进行垃圾回收;

#### 1.5.7.2.同步省略

在动态编译同步块的时候,JIT编译器可以借助逃逸分析来判断同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到其他线程.如果没有,那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步.这样就能大大提高并发性和性能.取消同步的过程就叫做锁消除;

~~~java
public void f(){
  Object key = new Object();
  synchronized(key){
    System.out.println(key);
  }
}
//代码中对key这个对象进行加锁,但是key对象的生命周期只在f()方法中,并不会被其他线程所访问,所以在JIT编译阶段就会被优化掉.优化完的代码如下:
public void f(){
  Object key = new Object();
  System.out.println(key);
}
~~~

#### 1.5.7.3.分离对象或标量替换

**标量**是指一个无法再分解成更小数据的数据,Java中的原始数据类型就是标量,相对的,那些还可以分解的数据叫做**聚合量**,Java中的对象就是聚合量,在JIT阶段,如果经过逃逸分析,发现一个对象不会被外界访问的话,那么经过JIT优化,就会**把这个对象拆解成其中包含若干个成员变量来代替**,这个过程就是标量替换;

`-XX:+EliminateAllocations`默认是打开的

```java
public staitc void alloc(){
	Point point = new Point(1,2);
  System.out.println(point.x+point.y);
}

Class Point{
  private int x;
  private int y;
}

//代码经过标量替换后,就会成为:
public staitc void alloc(){
	int x = 1;
  int y = 2;
  System.out.println(point.x+point.y);
}
```

Point这个聚合量经过逃逸分析后,并没有逃逸,就被替换成两个标量了,大大减少堆内存的占用,因为一旦不需要创建对象,就不再需要分配堆内存;

## 1.6.方法区

 ### 1.6.1.栈,堆,方法区的交互关系

![image-20210818162217195](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818162217195.png)

### 1.6.2.方法区的说明

尽管所有的方法区在逻辑上是属于堆的一部分,但是一些虚拟机的实现可能不会选择去进行垃圾收集或者进行压缩;HotSpot 虚拟机,方法区还有一个别名叫做Non-Heap(非堆),目的就是要和堆分开;所以方法区可以看作是一块独立于Java堆的内存空间;

我们可以把方法区是规范,HotSpot永久代和元空间是实现.JDK1.8之后采用的是元空间;

### 1.6.3.方法区大小的设置

* 元数据区大小可以使用参数`-XX:MetaspaceSize`和`-XX:MetaspaceSize`来设置初始元空间和最大元空间大小,默认值依赖于具体的平台;
* 与永久代不同,如果不指定大小,虚拟机会耗尽后所有可用的系统内存,如果发生溢出,`OutOfMemoryError:Metaspace`;
* `-XX:MetaspaceSize`设置初始的元空间大小,对于一个64位的服务器端JVM来说,默认的`-XX:MetaspaceSize`为21MB.这就是初始的高水位线,一旦触及,Full GC会被触发并卸载没有用的类(即这些类对应的类加载器不再存活),然后这个高水位线将会重置.新的高水位线的值取决于GC后释放了多少元空间,如果释放的空间不足,在不操作MaxMetaspaceSize时,适当提高该值,如果释放空间过多,则适当降低该值;
* 如果初始化的高水位设置过低,上述高水位线调整情况会发生很多次,通过垃圾回收器的日志可以看到Full GC多次调用,为了避免频繁GC,建议将`-XX:MetaspaceSize`设置为一个相对较高的值;

### 1.6.4.方法区的内部结构

方法区用于存储已被虚拟机加载的类型信息,常量,静态变量,即时编译器编译后的代码缓存等;

* 对每个加载的类型(class,interface,enum,annotation),JVM必须在方法区中存储一下类型信息:
  * 这个类型的完整有效名称(全名=包名.类名);
  * 这个类型直接父类的完整有效名(对于interface和Object都没有父类);
  * 这个类型的修饰符(public,abstract,final的某个子集);
  * 这个类型实现的直接接口的一个有序列表;
* 域(Field)信息
  * JVM必须在方法区中保存类型的所有域的相关信息以及域的声明顺序;
  * 域的相关信息包括:域名称,域类型,域修饰符;
* 方法(Method)信息
  * JVM必须保存所有方法的信息,包括声明顺序;
  * 方法名称;
  * 方法的返回类型;
  * 方法参数的数量,类型(按照顺序);
  * 方法的字节码(bytecodes),操作数栈,局部变量表大小(abstract和native方法除外);
  * 异常表(abstract和native方法除外);
    * 每个异常处理的开始位置,结束位置,代码处理在程序计数器中的偏移地址,被捕获的异常类的常量池索引;

### 1.6.5.class文件中的常量池

一个有效的字节码文件中除了包含类的版本信息,字段,方法以及接口等描述信息外,还包含一项信息就是常量池表(Constant Pool Table),包含各种字面量和对类型,域和方法的符号引用;

> 为什么需要常量池?
>
> 一个Java源文件中的类,接口,编译后产生一个字节码文件.而Java中的字节码需要数据支持,通常这种数据会很大以至于不能直接存到字节码里,换另一种方式,可以存到常量池,这个字节码包含了指向常量池的引用.在动态链接的时候会用到运行时常量池.
>
> ~~~java
> public class SimpleClass{
>   public void sayHeallo(){
>     System.out.println("hello");
>   }
> }
> ~~~
>
> 虽然上述代码的字节码文件很小,但是里面却用到了String,System,PrintStream以及Object等结构,如果代码多,引用到的结构会更多,这里就需要常量池了;

![image-20210818212655415](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818212655415.png)

上述方法中所有的`#`就是指向的常量池;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818213123252.png" alt="image-20210818213123252" style="zoom:50%;" />

常量池可以看成是一张表,虚拟机指令根据这张常量表找到要执行的类名,方法名,参数类型,字面量等信息;

### 1.6.6.运行时常量池

* 运行时常量池是方法区的一部分;
* **常量池表是Class文件的一部分,用于存放编译期生成的各种字面量与符号引用,这部分内容将在类加载后存放到方法区的运行时常量池中**;
* 在加载类和接口到虚拟机后,就会创建对应的运行时常量池;
* JVM为每个已加载的类型(类或接口)都维护一个常量池,池中的数据项像数组项一样,是通过索引访问的;
*  运行时常量池中包含多种不同的常量,包含编译期就已经明确的数值字面量,也包括到运行期解析后才能获得的方法或者字段引用.此时不再是常量池中的符号地址了,这里转换为真实的地址.运行时常量池,相对于Class文件常量池的另一重要特性是**具备动态性**,因为常量池中的数据一直在变动;
* 当创建类或接口的运行时常量池时,如果构造运行时常量池所需的内存空间超过了方法区所能提供的最大值,JVM抛出OOM;

### 1.6.7.方法区的演进

HotSpot方法区的变化:

* JDK1.6及之前:有永久代,静态变量存放在永久代上;
* JDK1.7 有永久代,逐步去除永久代,字符串常量池,静态变量移除,保存在堆中;
* JDK1.8 无永久代,类型信息,字段,方法,常量保存在本地内存的元空间,但字符串常量池,静态变量仍在堆中;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818235125263.png" alt="image-20210818235125263" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818235142361.png" alt="image-20210818235142361" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210818235243831.png" alt="image-20210818235243831" style="zoom:50%;" />

> StringTable为什么调整到堆中?
>
> JDK中将StringTable放到了堆空间中,因为永久代的回收效率很低,在Full GC的时候才会触发,而Full GC是老年代的空间不足,永久代不足的时候才会触发.这就导致了StringTable回收效率不高;而在开发中有大量的字符串被创建,回收效率低,导致永久代内存不足.放在堆中,能及时回收内存;
>
> 
>
> 静态变量是存储在堆中的;
>
> `private static person = new Person();` new出来的Person对象,肯定是放在堆中.person的静态变量从JDK1.7开始就放在堆中;

 ### 1.6.8.方法区的垃圾回收

方法区的垃圾收集主要回收内容:

* 常量池中废弃的常量;
* 不再使用的类型;

#### 1.6.8.1.常量的回收

方法区内常量池之中主要存放的两大类常量:**字面量**和**符号引用**.字面量比较接近Java语言层次的常量概念,比如文本字符串,被声明为final的常量值;而符号引用属于编译原理方面的概念,包括以下三类常量:

* 类和接口的全限定名;
* 字段的名称和描述符;
* 方法的名称和描述符;

只要常量池中的常量没有被任何地方引用,就可以被回收;

#### 1.6.8.2.不再使用类型的回收

判断一个类型是否属于"不再被使用的类"条件比较苛刻,需要**同时满足**三个条件:

* 该类所有的实例都已经被回收,也就是Java堆中不存在该类及其任何派生子类的实例;
* 加载该类的类加载器已经被回收,很难达成;
* 该类对应的Class对象没有在任何地方被引用,无法在任何地方通过反射反射访问该类的方法;

关于是否要对类型进行回收,HotSpot虚拟机提供了`-Xnoclassgc`参数进行控制,还可以使用`-verbose:class`以及`-XX:+TraceClass-Loading`,`-XX:+TraceClassUnLoading`查看类加载和卸载信息;

## 1.7.本地方法栈

本地方法栈用于管理本地方法的调用

* 本地方法栈是线程私有的;
* 允许被实现成股东或者是可动态扩展的内存大小;
* 本地方法是使用C语言实现;
* 具体的做法是Native Method Stack中登记native方法,在执行引擎执行时加载本地方法库;
* 在HotSpot JVM中,直接将本地方法栈和虚拟机栈合并起来;





