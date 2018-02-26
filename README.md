#  

--------
                                   《深入理解 Java 虚拟机--JVM高级特性与最佳实践》
--------

> ```
   关于这本书已经断断续续的看了好几遍了，使自己对jvm有了很深的理解，但是由于长时间的不用，对很多的功能点有所遗忘，特此写下这篇随手记，为以后的回忆与学习提供帮助，第一次写这种笔记，有种无从下手的感觉，后续持续改进中。。。
> ```

![深入理解 Java 虚拟机](picture/understandingofJVM.jpg)

### 1、关于java内存区域与内存溢出异常

​       下图为java运行时jvm将它所管理的内存划分的各个不同的数据区域，这些区域都有各自不同的用途，接下来，我会将这些区域的用途进行详细的介绍。

![java虚拟机运行时数据区](E:\git_repository\Understanding-of-JVM\picture\java虚拟机运行时数据区.jpg)

#### 1.1 程序计数器

​          程序计数器是一块较小的内存空间，它可以看做是当前线程所执行的字节码的行号指示器，字节码解释器就是通过它来确定java代码的执行顺序的，每一个线程都拥有独立的程序计数器，多线程是通过程序轮流切换处理器执行时间的方式来实现的，程序计数器可以保证每一个线程在切换后可以恢复到正确的执行位置。

#### 1.2 java虚拟机栈

​         许多coder将内存分为堆和栈，通过上图，我们知道，这种说法比较笼统，但是堆和栈确实占据了内存中大部分的空间，这个“栈”指的就是java虚拟机栈。

​        与程序计数器相同，java虚拟机也是线程所私有的，生命周期与线程相同，每个方法在执行的同时都会创建一个栈帧，用来存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用到执行完毕的过程就对应着一个栈帧在虚拟机栈中的入栈到出栈的过程。

![虚拟机栈用途](E:\git_repository\Understanding-of-JVM\picture\虚拟机栈用途.jpg) 

#### 1.3 本地方法栈

​         本地方法栈与java虚拟机栈作用相似，区别在与java虚拟机栈是为执行java方法而创建的栈，本地方法栈是为执行本地方法而创建的栈。

#### 1.4 java堆

​          java堆是在虚拟机启动的时候创建的，是被所有线程所共享的数据区域，它没有特殊的数据结构，只是内存的一块存储区域，占内存比例非常大，此区域的唯一目的就是存储创建的对象，java堆是垃圾收集器的主要管理区域，因此也被称为“GC堆”。

#### 1.5 方法区

​         与java堆相似，方法区也是被所有线程所共享的数据区域，它用于存储**已被**虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

#### 1.6 其他注意点

​        直接内存：直接内存不是java数据区中的内存，在jdk4中的nio（New Input/Output）中，引入了一种根据堆中DirectByteBuffer对象的引用来对这块直接内存进行操作的机制，直接内存是native函数库直接分配的堆外内存，这样可以避免java堆与Native堆之间数据的来回复制，因此，在机器上给java分配内存时，要注意有直接内存的存在。

### 2 Hot Spot 虚拟机对象的介绍

#### 2.1 对象的创建

​     1）虚拟机在遇到new这个指令时，首先会去方法区中的常量池检查该对象的类是否已经被加载过了，未加载过，则先进行类的加载、解析和初始化，然后为这个对象在堆上分配内存，内存的大小固定（大小通过它的类得知）。

​    2）内存分配完毕后，虚拟机将分配到的内存空间都初始化为零值（这就是为什么对象未初始化字段的话，基本类型为0，而引用类型为null）
```
package ObjectCreate;

public class DataInitTest {

public static void main(String[] args) {
	Data data = new Data();
	System.out.println(data.intA);
	System.out.println(data.doubleB);
	System.out.println(data.charC);
	System.out.println(data.floatD);
	System.out.println(data.booleanE);
	System.out.println(data.longF);
	System.out.println(data.byteG);
	System.out.println(data.data);
}

}

class Data{
	


int intA;
double doubleB;
char charC;
float floatD;
boolean booleanE;
long longF;
byte byteG;
Data data;


}
```
运行结果为：

![对象初始化结果](picture\对象初始化结果.jpg)

​     3）接下来，虚拟机要对对象头进行必要的设置，例如：这个对象时哪个类的实例，如何能找到类的元数据信息、对象的哈希码、对象的GC年龄等信息。

#### 2.2 对象的引用

​     在java虚拟机栈调用方法时，方法可能会用到引用变量，目前通过引用变量找到对象主流有两种方式：通过句柄访问对象、通过直接指针访问对象。

   1）通过句柄访问对象的话，引用中存的是句柄的地址，句柄中存放实例数据和类型数据的地址。

![通过句柄访问](E:\git_repository\Understanding-of-JVM\picture\通过句柄访问.jpg)

​    2）通过直接指针访问的话，引用中存放的就是对象地址，Hot Spot虚拟机使用的就是这种方式。

![通过直接指针访问对象](E:\git_repository\Understanding-of-JVM\picture\通过直接指针访问对象.jpg)

#### 2.3 JVM参数表

​    在有网络的前提下[点击此处](http://blog.csdn.net/coslay/article/details/49725899)查看JVM 参数表。

### 3 垃圾收集器与内存分配策略

#### 3.1 怎么判断一个对象已经“死亡”

​     可达性分析算法：这个算法的基本思路就是通过一系列的称为“GC roots”的对象作为起始点，从这些节点开始向下搜索，搜索走过的路径称为引用链，当一个对象到GC roots没有任何引用链相关联时，这个对象就是“死亡”的，下次进行GC时，若是该对象的finalize()方法没有重写或者已经被调用过，则该对象会被回收清理。

