# JVM_Second_Chapters_Note
## 深入理解Java虚拟机 JVM高级特性与最佳实践
## 第二章笔记
## Java内存区域
### 运行时数据区域
* 程序计数器
* 本地方法栈
* java堆
* 方法区
* 运行时常量池
* 直接内存
### 程序计数器(Program Counter Register)
```
程序计数器是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里，
字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程
恢复等基础功能都需要依赖这个计数器来完成。
虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，一个处理器都只会执行一条线程中的指令。
因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，我们称这类内存区域为
"线程私有"的内存。
如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址，如果正在执行的是Native方法，
这个计数器值为空。此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。
```
### 虚拟机栈
```
与程序计数器一样，栈也是线程私有的，它的生命周期与线程相同。栈描述的是Java方法执行的内存模型，每个方法的执行的同时都会
创建一个栈帧(Stack Frame)用于存储变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，
就对应着一个栈帧在栈中入栈到出栈的过程。
```
### 本地方法栈(Native Method Stack)
```
本地方法栈与虚拟机栈的作用非常相似，区别是虚拟机栈执行的是Java方法，也就是字节码，而本地方法栈则为虚拟机使用
到的Native方法服务。
```
### Java堆(Java Heap)
```
堆是虚拟机所管理的内存中最大的一块，且被所有线程共享的一块内存区域，在虚拟机启动时创建。
此内存唯一目的就是存放对象实例和数组。
堆也是垃圾收集器管理的主要区域，也被称做"GC堆"。由于现在的收集器基本都采用分代收集算法，
所以堆还可以细分为:新生代和老年代，
再细致一点有Eden空间、From Survivor空间、To Survivor空间等。
```
### 方法区(Method Area)
```
方法区与堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态常量即时编译后的代码等数据。
```
### 运行时常量池(Runtime Constant Pool)
```
运行时常量池是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于
存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。
运行期间也可能将新的常量放入池中，用的比较多的是String类的intern()方法。
```
### 直接内存(Direct Memory)
```
直接内存并不是运行时数据区的一部分，也不是java虚拟机规范中定义的内存区域。
在JDK1.4中新加入了NIO(New Input/Output)类，引入了一种基于通道(Channel)与缓冲区(Buffer)的I/O方式，它可以使用
Native函数直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。
这样能显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。
```
---
### HotSpot虚拟机对象探秘
* 对象的创建
```
当虚拟机遇到一条new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表
的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。
类加载检查通过后，接下来将为新生对象分配内存。对象所需内存的大小在类加载完成后便可确定。
内存的分配方式有两种，一种为指针碰撞(Bump the Pointer)，另一种为空闲列表(Free Lisst)。选择哪种与java堆内存收集器
是否带有压缩整理功能决定。  
内存分配成功后，虚拟机需要将分配到内存空间都初始化为零值。这一步保证了对象的实例字段在Java代码中可以不  
赋初始值就可以直接使用。
接下来，虚拟机要对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、
对象的GC分代年龄等信息。这些信息存放在对象头(Object Header)之中。
上面工作都完成后，从虚拟机的视角来看，一个新的对象已经产生了，但从Java程序的视角来看，对象创建才刚刚开始，
因为<init>方法还没有执行，所有的字段都还是零。其实就是还没有执行对象的构造方法。
```
* 对象的内存布局
```
对象在内存中的存储布局分为3块区域，对象头(Header)、实例数据(Instance Data)和对齐填充(padding)。
```
1. 对象头
```
对象头包括两部分信息，
第一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、
偏向时间戳等。
第二部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
如果对象是数组，那对象头中还必须有一块用于记录数组长度的数据。
```
2. 实例数据
```
实例数据是真正存储对象有效信息的区域，也是代码中定义的各种类型的字段内容。
```
3. 对齐填充
```
对齐填充不是必然存在的，也没有特别的含义，仅仅起着占位符的作用。
```
* 对象的访问定位
```
Java程序需要通过栈上的reference数据来操作堆上的具体对象。目前访问堆上具体对象有两种方式。
句柄和直接指针两种。
```
1. 句柄
```
使用句柄访问的话，那么Java堆中将会划分出一块内存来作为句枘池，reference中存储的就是
对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。
优点：
reference中存储的是稳定的句柄地址，在对象被移动(垃圾收集时移动对象非常普遍)时只会改变
句柄中的实例数据指针，而reference本身不需要修改。
```
2. 直接指针
```
如果使用直接指针访问，那么reference中存储的直接就是对象地址。
优点：
速度快，节省了一次指针定位时间开销。
```
