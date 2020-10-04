## JVM内存区域划分

JVM运行时数据区分为：堆、方法区、栈（虚拟机栈、本地方法栈）、程序计数器。

<img src="https://upload-images.jianshu.io/upload_images/2179030-f81a3bf0df216749.png?imageMogr2/auto-orient/strip|imageView2/2/w/813/format/webp" alt="JVM内存结构" style="zoom:67%;" />

## 1. 程序计数器 Program Counter

- 线程私有，是**当前线程**的字节码行号指示器。
  - 如果线程执行 Java 方法，指示字节码指令的地址（行号指示器）；
  - 如果执行的是 Native 方法，计数器值为 `Undefined`。
- PC 中无异常发生。

> 程序计数器保证了线程挂起和线程再次获得CPU使用权时，代码的运行顺序。

## 2. 虚拟机栈（线程栈）Java Virtual Machine Stacks

### 2.1 用途

- 虚拟机栈为 Java 方法服务。
- **线程私有**，每个线程都维护一个与线程同时创建的虚拟机栈，线程调用的每个方法对应一个**栈帧**，栈帧中储存了一个方法的状态信息：
  -  **局部变量表**、**操作数栈**、**动态链接**、**方法出口** 等。
- 可能发生的异常：
  -  `StackOverflowError`：方法调用的深度超过了虚拟机栈深度。
  - `OutOfMemoryError`：虚拟机栈随着内容的增加会**动态扩展内存**，扩展到一定程度无法申请到足够的内存时抛出该异常。
- 使用 `-Xss` 设置栈大小，通常几百K就够用了。由于栈是线程私有的，线程数越多，占用栈空间越大。

![image](https://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/D724E9EB4C4A4BD4ABC5E3DCA1DCF0D5/8856)

### 2.2 局部变量表

局部变量表用于储存数据：
- **基本数据类型**
- **对象引用**（包括 **方法参数** 和 **局部变量**）

#### 2.2.1 变量槽 Variable Slot

局部变量表的最小单位是 **变量槽（Variable Slot）**，每个变量槽 32 位（4字节）。通过 index 的方式直接索引。
- 普通方法 0 号位存放 `this` 指针，`static` 方法没有 `this`，所以直接存其他数据。
- Slot 可以复用，当局部变量作用域失效时，可以被其他数据覆盖。

### 2.3 操作数栈

存在于虚拟机栈中，用于辅助计算，存放用于计算的操作数。

## 3. 本地方法栈

本地方法栈为 native 方法服务，其他与虚拟机栈类似。

native 方法比如：`String.extern()` 方法。

异常：

- StackOverflow
- OutOfMemory

> hot spot 虚拟机中的 虚拟机栈 和 本地方法栈 是合在一起的。

## 4. Java 堆 Heap

### 4.1 堆的用途

- 堆在 JVM 启动时被创建，用于存放所有的对象实例、数组、String 等。
- Java 堆是 JVM 内存管理的最大区域，是垃圾回收的主要场所。
- Java 堆是**线程共享**的。

### 4.2 堆的结构

Java 堆按垃圾回收区域分为 **年轻代** 和 **老年代** ，具体：
<img src="https://img-blog.csdnimg.cn/2019041019553012.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70" alt="java堆a" style="zoom:80%;" />

![image](https://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/18FF35D98E7B4C84A637F1EDB0CD91F5/8810)

- 年轻代存储新创建的对象。当年轻内存占满后，会触发 `Minor GC`，清理年轻代内存空间。
- 老年代存储长期存活的对象和大对象。年轻代中存储的对象，经过多次GC后仍然存活的会移动到老年代中进行存储。老年代空间占满后，会触发`Full GC`，清理整个堆空间，包括年轻代和老年代。如果`Full GC`之后，堆中仍然无法存储对象，就会抛出`OutOfMemoryError`异常。

### 4.3 Java 堆设置常用参数

- -**Xms**：设置Java应用程序启动时的初始堆大小
- -**Xmx**：设置Java应用程序能获得的最大堆大小
- -Xss：设置线程栈的大小
- -XX:MinHeapFreeRatio：设置堆空间最小空闲比例。当对空间的空闲内存小于这个数值时，JVM便会扩展堆空间
- -XX:MaxHeapFreeRatio：设置堆空间的最大空闲比例。当堆空间的空闲内存大于这个数值时，便会压缩堆空间，得到一个较小的堆
- -XX:NewSize：设置新生代的大小
- -XX:NewRatio：设置老年代与新生代的比例，它等于老年代大小除以新生代大小
- -XX:SurviorRatio：新生代中eden区与survivior区的比例
- -XX:MaxPermSize：设置最大的持久区的大小
- -XX:PermSize：设置永久区的初始值
- -XX:TargetSurvivorRatio：设置survivior区的可使用率。当survivior区的空间使用率达到这个数值时，会将对象送入老年代


## 5. 方法区 Method Area

- 存放已被虚拟机加载的 **类信息**、**常量**、**static变量**、**即时编译后的代码 **等。
  - 存放每个类的结构信息
  - 类属性
  - 类的方法
  - 方法和类的构造函数的代码（包括类实例在初始化、接口初始化中使用的特殊方法）
- **线程共享**。
- **常量池**是方法区的一部分。
  - **class文件常量池**
  - **运行时常量池**
- 在JDK 1.7及以前，方法区是堆的一个逻辑部分；为了与堆区分，被称为“非堆”或“永久代”。

<img src="https://images2018.cnblogs.com/blog/1425453/201808/1425453-20180801203110267-64138529.png" alt="方法区组成a" style="zoom:70%;" />

### 5.1 class 文件常量池

class 文件常量池存在于 .class 字节码文件中。

存储编译器生成的各种 **字面量** 和 **符号引用** 。
- **字面量**：文本字符串、final 常量、基本类型常量的值
- **符号引用**：
    - 类和接口的全限定名：用于在运行时类加载过程“解析”阶段得到直接引用。
    - 方法的名称和描述符：描述符即参数类型和返回值类型。
    - 字段的名称和描述符：包括类变量（static）、实例变量。


### 5.2 运行时常量池

- 运行时常量池是**线程共享**的。
- 存储 Java class文件常量池中的**符号信息**。在类加载时，第一步是“加载”，包括三步：（1）通过类全限定名来获取二进制字节流；（2）把字节流所代表的**静态存储结构**转化为方法区的**运行时数据结构**；（3）方法区中生成Class对象，作为该类的各种数据访问入口。其中，**第（2）步中就包含了class常量池中进入运行时常量池的过程。**
- 存放**直接引用**。类加载的“解析”阶段会将符号引用所翻译出来的直接引用(直接指向实例对象的指针)存储在 运行时常量池 中。
- 常量不一定编译时才能产生，**运行时产生的常量**会放入运行时常量池。


### 5.3 字符串常量池（看Reference 3、4）

- JDK 1.7 之前，字符串常量池存在于运行时常量池（方法区）内；

- JDK 1.7 把其移到了Java 堆中。

- JDK 1.8 中，字符串常量池被移到了元空间，独立于堆。

常量池避免了频繁的创建和销毁对象而影响系统性能，其实现了对象的共享。

以`String`类为例，可以通过反编译验证：

```java
String str1 = "abcd";               // 放在 常量池 中
String b = "a" + "true";            // 编译期就计算完，然后放在 常量池

// 通过 new 的方式创建String，其直接引用对象是在堆中，但仍涉及字符串池。
// 具体：先判断字符串常量池内有无该字符串：
// 如果有，在堆中复制一个副本，把副本地址给引用；
// 如果没有，先在字符串池中创建一个，然后复制副本到堆，然后把副本地址给引用。
String str1 = new String("abcd");
```

> `String.intern()` 方法会返回字符串在字符串常量池中的引用。

### 5.4 基本类型包装类的常量池

整型的包装类在对应值小于等于127时才可使用对象池。这里常量池中缓存的是包装类对象，而不是基本数据类型。

涉及包装类的缓存技术，看我的基本类型那块笔记。

### 5.5 方法区随 JDK 版本的变化

![方法区的变化](https://img-blog.csdn.net/20180807233340374?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1eXV5YW5nNjY4OA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### JDK 1.7

JDK 1.7时的Hotspot VM，把方法区中的静态变量、字符串常量池等移到了堆内存。

#### JDK 1.8

JDK 1.8中，方法区被替换为**元空间 Meta Space**，它没有占用堆内存，而是直接占用本地内存。

方法区的参数取代为：

- -XX:MetaspaceSize
- -XX:MaxMetaspaceSize

## Reference 

[一文搞懂JVM内存结构](https://blog.csdn.net/rongtaoup/article/details/89142396)

《深入理解Java虚拟机》

[JVM常量池浅析](https://www.jianshu.com/p/cf78e68e3a99)

[JVM中的常量池详解](https://www.cnblogs.com/superyc/p/9975254.html)