## 一 JVM内存模型和内存分配

### 1.1 介绍下Java内存区域（运行时数据区）

![img](pics\131449384023660.png)

Java虚拟机将内存区域划分为堆、方法区、程序计数器、虚拟机栈、本地方法栈，其中堆和方法区是线程共享的，程序计数器、虚拟机栈、本地方法栈是属于线程私有的。

JVM初始运行的时候都会分配好**Method Area（方法区）**和**Heap（堆）**，而JVM 每遇到一个线程，就为其分配一个**Program Counter Register（程序计数器）**, **VM Stack（虚拟机栈）和Native Method Stack （本地方法栈），**当线程终止时，三者（虚拟机栈，本地方法栈和程序计数器）所占用的内存空间也会被释放掉。这也是为什么我把内存区域分为线程共享和非线程共享的原因，非线程共享的那三个区域的生命周期与所属线程相同，而线程共享的区域与JAVA程序运行的生命周期相同，所以这也是系统垃圾回收的场所只发生在线程共享的区域（实际上对大部分虚拟机来说知发生在Heap上）的原因。

> 程序计数器

程序计数器是一块较小的内存区域，作用可以看做是当前线程执行的字节码的位置指示器。分支、循环、跳转、异常处理和线程恢复等基础功能都需要依赖这个计数器来完成。

> Java 虚拟机栈

Java 虚拟机栈描述的是Java方法执行的内存模型，每个方法执行的同时创建帧栈（Strack Frame）用于存储局部变量表（包含了对应的方法参数和局部变量），操作栈（Operand Stack，记录出栈、入栈的操作），动态链接、方法出口等信息，每个方法被调用直到执行完毕的过程，对应这帧栈在虚拟机栈的入栈和出栈的过程。

> 堆

此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。

> 方法区

用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

> 运行时常量池

运行时常量池是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池表（用于存放编译期生成的各种字面量和符号引用）





## 二 垃圾回收算法

Java 的自动内存管理主要是针对对象内存的回收和对象内存的分配。同时，Java 自动内存管理最核心的功能是 **堆** 内存中对象的分配与回收。

### 2.1 如何判断对象是否死亡

> 引用计数法

给对象添加一个引用计数器，每当有一个地方引用它，计数器加1；当引用失效，计数器就减1；任何时候计数器为0的对象就是不可能再被使用的。

> 可达性分析算法

这个算法的基本思想就是通过一系列的称为“GC Roots”的对象作为起点，从这些节点开始向下搜索，节点所走过的路径称为引用链，当一个对象到GC Roots 没有任何引用链相连的话，则证明此对象是不可用的。

可作为 GC Roots 的对象包括下面几种:

- 虚拟机栈(栈帧中的本地变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 所有被同步锁持有的对象

### 2.2 垃圾收集算法

> 标记-清除算法

该算法分为“标记”和“清除”阶段：首先标记出所有不需要回收的对象，在标记完成后统一回收掉所有没有被标记的对象。它是最基础的收集算法，后续的算法都是对其不足进行改进得到。这种垃圾收集算法会带来两个明显的问题：

1. **效率问题**
2. **空间问题（标记清除后会产生大量不连续的碎片）**

> 标记-复制算法

为了解决效率问题，“标记-复制”收集算法出现了。它可以将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

> 标记-整理算法

根据老年代的特点提出的一种标记算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

> 分代收集算法

当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将 java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

**比如在新生代中，每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。**







## 三 类加载机制

### 3.1 类加载的过程

系统加载Class类型的文件主要三步：加载->连接->初始化。连接过程又可分为三步：验证->准备->解析。

> 加载

类加载过程的第一步，主要完成下面 3 件事情：

1. 通过全类名获取定义此类的二进制字节流
2. 将字节流所代表的静态存储结构转换为方法区的运行时数据结构
3. 在内存中生成一个代表该类的 `Class` 对象，作为方法区这些数据的访问入口

> 验证

验证分成：文件格式验证、元数据验证、字节码验证和符号引用验证

> 准备

**准备阶段是正式为类变量分配内存并设置类变量初始值的阶段**，这些内存都将在方法区中分配。

> 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

> 初始化

初始化阶段是执行初始化方法 `<clinit> ()`方法的过程，是类加载的最后一步，这一步 JVM 才开始真正执行类中定义的 Java 程序代码(字节码)。