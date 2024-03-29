---
title: Java 类加载机制
date: 2020-9-26 9:59:50
tags: JVM
categories: JVM
mathjax: true
---

# Java 类加载机制总结

类加载是把字节码 .class 文件加载到内存里，从而生成对应类的 Class 对象，同一个类只有一个 Class 对象。当该类需要被实例化的时候，即用 new 关键字来创建对象时，JVM 会去获取该 Class 对象的信息。

## 1. Class 文件

.class 字节码文件包含：
- **基本的描述信息**：类版本、字段、方法、接口等
- **静态常量池**：存储编译器生成的各种 **字面量** 和 **符号引用** 。
    - **字面量**：文本字符串、final 常量、基本类型常量的值，简单来说，就有由字母、数字等构成的字符串或数值。
    - **符号引用**：
        - **类和接口的全限定名**：用于在运行时类加载过程“解析”阶段得到直接引用。
        - **方法的名称和描述符**：描述符即 **参数类型** 和 **返回值类型**。
        - **字段的名称和描述符**：包括**类变量（static**）、**实例变量**。

### 1.1 Class 文件的结构

Class 文件中只有两种数据类型： **无符号数**、**表**。

- 无符号数：u1、u2、u4、u8 分别表示 1字节、2字节、4字节、8字节的无符号数。
- 表：由 0 个或多个大小可变的项组成，一个类就相当于一个表。

> **Class 文件中没有任何对齐和填充的说法，所有数据都按照特定的顺序紧凑的排列在 Class 文件中**。
>
> 不像堆中的对象内存一样，会填充为 8 字节的整数倍。

~~~java
ClassFile {
    u4             magic;				// 魔数：判断该文件是否为 JVM 使用的字节码文件
    u2             minor_version;		// 次版本号：JDK 12 前未使用
    u2             major_version;		// 主版本号：标识 JDK 版本等，向下版本兼容
    u2             constant_pool_count;	// 常量池数量
    cp_info        constant_pool[constant_pool_count-1];// 常量池信息
    u2             access_flags;				// 访问标志：标识类或接口的访问级信息
    u2             this_class;// 类索引
    u2             super_class;// 父类索引
    u2             interfaces_count;//接口数(2位，所以一个类最多65535个接口)
    u2             interfaces[interfaces_count];//接口索引 
    u2             fields_count;//字段数
    field_info     fields[fields_count];		// 字段表集合：描述接口或类中的变量，包括类变量和实例变量
    u2             methods_count;//方法数
    method_info    methods[methods_count];//方法集合
    u2             attributes_count;//属性数
    attribute_info attributes[attributes_count];//属性表集合
}
~~~
![](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/40655667A3834A1CBA07F8C75083F0C5/9802)

### 1.2 静态常量池

常量池中存放两大类变量：**字面量** Literal 和 **符号引用** Symbolic Reference。

- 字面量：文本字符串、final 常量等。
- 符号引用：包信息、类和接口的全限定名、字段的名称与描述符、方法的名称与描述符、方法句柄和方法类型、动态调用点和动态常量。

在 Class 文件中不会保存各个方法、字段的最终内存布局信息，因此这些字段、方法的符号引用不经过运行期转换的话无法得到真正的内存入口地址，也就无法直接被虚拟机使用。当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。

使用命令行查看 class 文件常量池：

```shell
javap -v 文件名.class
```

## 2. 类加载机制

Java中的类加载、连接和初始化都是在**运行时**完成的，Java的动态扩展的特性就依赖于运行时的动态加载和动态连接。

<img src="http://upload-images.jianshu.io/upload_images/12219352-2d61700695ba47c6.png?imageMogr2/auto-orient/strip|imageView2/2/w/662/format/webp" alt="类加载过程a" style="zoom:80%;" />

图中前五部分被称为类加载。包括：**加载、验证、准备、解析、初始化**。

类加载的三个大阶段：
- 加载
- 连接
- 初始化

### 2.1 类加载的前提：编译

在 JVM 运行之前，java 代码被编译器 javac 编译成 .class 字节码文件。一些类加载时需要用到的编译期知识：

- 编译期间的常量：
    - 编译期间，知道确定值的常量会直接被加入到常量池中，不会经历类加载过程。
    - 不知道确定值的常量，则会在**运行时**对该常量所在的类进行初始化。
        - 如果该`static final`类型是 **基本类型** 或者 **字符串**，则会被编译器标记成`ConstantValue`，后面运行时类加载中的 “**准备**” 阶段会对其进行直接初始化。
- 编译时，编译器会按照代码顺序自动收集类中所有的 **静态变量的赋值动作** 和 **静态语句块** 中的语句，合并产生`<clinit>()`方法。如果没有符合要求的语句，则不生成。
    - （接口因为也有static，所以也会生成`<clinit>()`方法）

### 2.2 加载 Loading

加载过程主要做三件事：

1. 通过一个类的全限定名来获取定义此类的**二进制字节流**。
   - 二进制字节流的来源：.class 文件、jar 包等压缩包、网络中获取、动态代理动态生成 .class 文件等
2. 将这个字节流所代表的**静态存储结构**转化为方法区的**运行时数据结构**。
3. 在内存中生成一个代表这个类的 `java.lang.Class` 对象，作为该类的各种数据的访问入口。
   - Class类是特殊的对象，不一定分配在堆中，Hot Spot 虚拟机分配在方法区中（1.7之前）。

#### 2.2.1 类加载器

引用类型包括类、接口、数组类、泛型参数。
- 泛型参数会在编译期间进行泛型擦除，不涉及运行时。
- 数组类没有字节流，不通过类加载器进行加载，而是虚拟机直接创建。但是若数组的**组件类型是引用类型**，则递归地去用类加载器加载其组件类型。
- 类和接口通过类加载器去获取字节流。

<img src="http://upload-images.jianshu.io/upload_images/13898040-92981cbcdd89a0fe.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp" alt="类加载器a" style="zoom:50%;" />

除了“启动类加载器”，其他类加载器都是`java.lang.ClassLoader`的子类。

- **启动类加载器**：C++实现，没有对应的Java对象。用于加载最基础、最重要的类，如JRE的 /lib 目录下的 jar 包中的类。
- **扩展类加载器**：负责加载次要但通用的类，如JRE的 /lib/ext 目录下 jar 包中的类。
- **应用类加载器**：负责加载应用程序路径 `classpath` 下的类。

#### 2.2.2 双亲委派机制

> 当一个类加载器收到类加载任务，它自己首先**不会**自己主动去加载这个类，而是先交给其**父类加载器**去尝试加载，直到传递到顶层的**启动类加载器**。只有当父类加载器无法完成加载任务时，才会尝试**子类加载器**执行加载任务。
>
> **双亲委派机制保证了**：对同一个类，不管是哪个加载器加载这个类，最终都是委托给可能的最顶层的类加载器进行加载，来保证使用不同的类加载器都会得到相同的 Object 对象。同时可以防止核心 API 库被随意篡改。

<img src="http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/2A8C9B8B5D744E08ABAEA94B4CD8B3AB/9138" alt="a" style="zoom:70%;" />

- **启动类加载器** (Bootstrap ClassLoader)：负责加载 `$JAVA_HOME\lib` 下的类或者被参数 `-Xbootclasspath` 指定的能被虚拟机识别的类(通过jar名字识别，如：rt.jar)，启动类加载器**由 Java 虚拟机直接控制**，开发者不能直接使用启动类加载器。
  - 启动类加载器没有子类，但是在**逻辑上**当扩展类加载器会将收到的类加载请求传递给启动类加载器来进行优先加载。

- **扩展类加载器** (Extension ClassLoader)：负责加载 `$JAVA_HOME\lib\ext` 下的类，或者被 `java.ext.dirs` 系统变量指定路径中的所有类库( `System.getProperty(“java.ext.dirs”)` )，开发者可以直接使用这个类加载器。

- **应用程序类加载器** (Application ClassLoader)，负责加载 `$CLASS_PATH` 中指定的类库。开发者能直接使用这个类加载器。
  - 正常情况下如果没有自定义类加载器，一般用的就是这个类加载器。

- **自定义类加载器**：可以通过继承 `java.lang.ClassLoader` 来自定义类加载器,一般我们都选择继承 `URLClassLoader` 来进行适当的改写就可以了。

> 可以通过继承  `java.lang.ClassLoader` ，并重写其中的 `loadClass()` 方法来破坏双亲委派机制。



### 2.3 连接 Linking

#### 2.3.1 验证 Verification

确保 class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机的自身安全。

1. **文件格式验证**：比如说是不是以魔数开头，jdk 版本号的正确性等等。
2. **元数据验证**：比如说类中的字段是否合法，是否有父类，父类是否合法等等。
3. **字节码验证**：确定程序语义是合法的、符合逻辑的。
4. **符号引用验证**：类对自身以外（常量池中的各种符号引用）的信息进行匹配性校验，目的是确保解析动作能正常执行。

#### 2.3.2 准备 Preparation

为类或接口的**静态字段** `static`（类变量、静态常量）分配内存（在**方法区**中），并初始化为**默认值**。

- 若不是 `static final`，该初始值指的是类型对应的**默认初始值**（如下表），并非开发者对变量赋值的初值。
- 若为 `static final`，即为静态常量，则直接初始化为赋予的初始值。
  - 常量在**编译期**会被添加常量标志 `ConstantValue`，JVM 以此来判断是不是常量。

![默认初始值](http://upload-images.jianshu.io/upload_images/12219352-1c3e74652ae829c4.png?imageMogr2/auto-orient/strip|imageView2/2/w/737/format/webp)

#### 2.3.3 解析 Resolution

解析是虚拟机把 class 文件常量池内的**符号引用**替换为运行时的**直接引用**的过程。直接引用是**直接指向目标的指针**等，前提是引用的目标已经在内存中了（这也意味着解析阶段必须在准备阶段之后，因为准备阶段才正式开始分配内存）。

- **符号引用**：符号引用与 JVM 的内存布局无关。编译期不知道实际内存地址，所以采用字面量形式的符号引用。
    - 符号可以是任意形式的字面量，该符号可以唯一定位到引用的目标。
- **直接引用**：直接引用与 JVM 的内存布局有关。直接指向目标的指针、相对偏移量、或是一个能间接定位到目标的句柄。
    - 直接引用的目标一定是已经加载到内存中的内容。

> 因为 Java 支持动态绑定，所以有些引用要等到具体使用的时候才会知道指向，所以**解析可以在初始化之后进行**。

解析动作主要针对的是类或者接口、字段、类方法、方法类型、方法句柄和调用点限定符 7 类**符号引用**。

##### 2.3.3.1 类或接口的解析

若一个类或接口的符号引用未被解析，则对其的解析分为以下几步：
1. 如果**非数组类型**，则通过全限定名给类加载器去加载该类。
    - 这个过程涉及验证过程，所以可能会触发其他相关类的加载
2. 如果是**数组类型**，且元素类型是对象，则按 `1` 规则加载其元素类型的类。
3. 解析完成前需要进行符号引用验证，确认当前调用这个符号的类是否有对该符号引用的访问权限。否则抛出 `java.lang.NoSuchFieldError` 异常。
4. 如果上面的步骤没有出现异常，则该符号引用会在虚拟机中产生直接引用。

##### 2.3.3.2 字段的解析

对字段的解析首先需要对其**所属的类**解析，因为字段是属于类的。

对字段的解析分为以下几步：
1. 如果该字段的符号引用只包含**简单名称**和**字段描述符**都与目标相匹配的字段，则返回这个字段的直接引用，解析结束；
2. 否则，如果该符号引用所属的类实现了**接口**，将会按照继承关系从下往上递归搜索各个接口和接口的父接口，如果在接口中包含了简单名称和字段描述符都与目标相匹配的字段，那么直接返回这个字段的直接引用，解析结束；
3. 否则，如果该符号所在的类不是 `Object` 类的话，将会按照继承关系从下往上递归搜索其**父类**，如果在父类中包含了简单名称和字段描述符都相匹配的字段，那么直接返回这个字段的直接引用，解析结束;
4. 否则，解析失败，抛出 `java.lang.NoSuchFieldError` 异常

成功解析符号引用为直接引用的字段，会进行**权限验证**，如果不具备对字段的访问权限，则抛出 `java.lang.NoSuchFieldError` 异常。

##### 2.3.3.3 类方法解析

类方法是指类中 `static` 修饰的方法。解析类方法的前提是解析其所在的类。

对类方法的解析步骤分为：
1. 类方法和接口方法的符号引用是分开的，所以如果在**类方法表**中发现 `class_index`（类中方法的符号引用）的索引是一个接口，那么会抛出 `java.lang.IncompatibleClassChangeError` 异常;
2. 如果 `class_index` 的索引确实是一个类，那么在该类中查找是否有简单名称和描述符都与目标字段相匹配的方法，如果有的话就返回这个方法的直接引用，解析结束；
3. 否则，在该类的**父类**中递归查找是否具有简单名称和描述符都与目标字段相匹配的字段，如果有，则直接返回这个字段的直接引用，解析结束；
4. 否则，在这个类的**接口**以及接口的父接口中递归查找，如果找到的话就说明这个方法是一个**抽象类**，查找结束，返回 `java.lang.AbstractMethodError` 异常;
5. 否则，查找失败，抛出 `java.lang.NoSuchMethodError` 异常。

##### 2.3.3.4 接口方法解析

1. 如果在**接口方法表**中发现 `class_index` 的索引是一个类而不是一个接口，会抛出 `java.lang.IncompatibleClassChangeError` 的异常;
2. 否则，在该接口方法的所属的接口中查找是否具有简单名称和描述符都与目标字段相匹配的方法，如果有的话就直接返回这个方法的直接引用;
3. 否则，在该接口的父接口中查找，直到 `Object` 类，如果找到则直接返回这个方法的直接引用;
4. 否则，查找失败。

接口所有方法都是 `public` 方法，所以不存在访问权限问题。

### 2.4 初始化 Initialization

初始化阶段是真正开始执行字节码进行赋值操作，会把准备阶段的默认值替换为真正的初始值。

编译时，编译器会按照代码顺序自动收集类中所有的 **静态变量的赋值动作** 和 **静态语句块** 中的语句，合并产生`<clinit>()`方法。初始化过程就是`<clinit>()`执行的过程。初始化完成后，类即成为可执行状态。

#### 2.4.0 <clinit>() 方法与 <init>() 方法

`<clinit>()` 方法是对静态域的初始化操作，它的执行时期是类加载的初始化阶段。

- 在一个类的`<clinit>()`方法执行前，其父类的`<clinit>()`方法已经执行完毕。因此第一个执行该方法的类肯定是`java.lang.Object`。
- 如果一个实现类或子接口实现了某父接口，则不需要先执行父接口的`<clinit>()`方法。当父接口中的某些属性被使用到的时候才会触发父接口的初始化。
- JVM 会通过加锁来保证类的`<clinit>()`方法只被执行一次。

`<init>()` 方法是对象实例的构造器方法。

- 当通过 new 关键字去创建对象时触发，具体阶段是在对象实例的内存分配完毕之后，并初始化了默认值，然后会调用对象的构造器 Constructor。

#### 2.4.1 初始化何时被触发

JVM 规定了 5 种情况必须进行**立即**初始化，也被称为**主动引用**。

1. 当虚拟机启动时，**主类**（main 方法所在的类）被初始化。
2. `new` **实例化**一个类对象时；或者调用或设置某类的**静态方法**或**静态字段**（除**编译期常量**）时，初始化所在的类。
    - 编译期常量是指编译期就能确定值的 `final` 修饰的字段。
3. 当初始化子类时，如果父类还没初始化，则先触发**父类的初始化**。
    - 若类实现了某定义了 `default` 方法的接口（1.8及之后），则该类的初始化会触发该接口的初始化。
4. 使用反射 API 对某个类进行反射调用时。
5. JDK 1.7 开始提供的**动态语言支持**，如果一个 `java.lang.invoke.MethodHandle` 实例解析的结果`REF_getStatic`，`REF_putStatic`，`REF_invokeStatic` 的方法句柄对应的类没有被初始化，需要触发其初始化。

##### 2.4.2.1 对于父接口

当接口被初始化时，不要求其父接口全部初始化，只有真正使用到父接口时才会触发父接口的初始化。如使用到了父接口中定义的常量等。

##### 2.4.2.1 对于数组

~~~java
Object[] arr = new Object[10];
~~~

构造数组对象和直接构造对象是用过不同的字节码来实现的，创建数组对象是通过 `newarray` 指令来实现，所以并不会初始化 Object 对象。


#### 2.4.2 被动引用不触发初始化

1. 通过**数组**定义类 A 的引用，不会触发该类 A 的初始化 `A[] = new A[10]`;
2. 通过子类访问父类的**静态域**时，只有父类会被初始化（即真正声明这个域的类）。
3. 引用常量不会导致类的初始化，因为常量在编译期就被加入了常量池。

## 3. Class 类

类加载分为三个阶段：
- 加载
- 连接
- 初始化

Class 类在类加载的 **加载** 阶段被加载到内存中。

### 3.1 获取 Class 类导致类加载？

通过**反射调用**和继承自 `Object` 类的方法的方式来获取 Class 类对象时，会触发完整的类加载：
- `Class.forName("com.zgxh.Zgxh")`
- `instance.getClass()`

通过**字面量**的形式来获取 Class 类对象的引用时，只会触发到类加载的加载过程，把 `Class` 加载进来，而不会触发初始化。
- `Zgxh.class`

## ClassLoader 详解

![ClassLoader 继承图](http://note.youdao.com/yws/public/resource/4abadcd0262eda3859a001aa3e1fcc28/xmlnote/CE379F1C3202473DAF88DBF2354530F6/20145)

`ExtClassLoader` 和 `AppClassLoader` 都继承自 `URLClassLoader`。

顶层类加载器是 `ClassLoader`。

### 一些方法分析

#### loadClass()

`loadClass()` 方法用于实现具体的加载逻辑：

当类加载请求来临时，先从缓存中查找该 class 对象，如果有了，就不加载了。

如果没有，则通过双亲委派模型去交给父类来加载，如果没有父类的话，则交给 `BootstrapClassLoader` 去加载。 

如果都没有找到对应的类加载器，则使用 `findClass()` 去加载。

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
{
  synchronized (getClassLoadingLock(name)) {
      // 先从缓存查找该class对象
      Class<?> c = findLoadedClass(name);
      if (c == null) {
          long t0 = System.nanoTime();
          // 尝试通过双亲委派机制来交给父类加载器去加载
          try {
              if (parent != null) {
                  c = parent.loadClass(name, false);
              } else { // 如果没有父类，则委托给启动加载器去加载
                  c = findBootstrapClassOrNull(name);
              }
          } catch (ClassNotFoundException e) {
              // ClassNotFoundException thrown if class not found
              // from the non-null parent class loader
          }

          if (c == null) {
              // 如果都没有找到，则通过自定义实现的findClass去查找并加载
              c = findClass(name);

              // this is the defining class loader; record the stats
              sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
              sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
              sun.misc.PerfCounter.getFindClasses().increment();
          }
      }
      // 是否需要在加载时进行解析
      if (resolve) {
          resolveClass(c);
      }
      return c;
   }
}
```

#### findClass()

顶级类 `ClassLoader` 中 `findClass()` 方法是空方法，直接抛 `ClassNotFoundException`。

可以通过改写 `findClass()` 方法来自定义自己的类加载逻辑。
- 前提是在调用父类加载器加载失败时才会用到该方法进行加载。

`findClass()` 中一般要实现自己的获得 class 对应的字节流等方法。
- `URLClassLoader` 类实现了 `findClass()` 等辅助方法。在自定义类加载器时通过继承 `URLClassLoader` 可以省略一些逻辑。

### 自定义文件类加载器、网络类加载器、热部署类加载器

参考 [深入理解java类加载器](https://blog.csdn.net/javazejian/article/details/73413292)

### 线程上下文类加载器

区别于 `API （Application Programming Interface`），Java 中的 `SPI （Service Provider Interface`），对外提供接口，供第三方来实现。

`SPI` 接口，如 `JDBC`，`JNDI` 等属于核心库，一般存在于 `rt.jar` 中，这个 `jar` 归 `Bootstrap` 类加载器负责。但开发中， `SPI` 的第三方实现代码在 `classpath` 下，不归 `Bootstrap` 类加载器负责，所以需要委派给特殊的类加载器-- **线程上下文类加载器**。
- 通过 `java.lang.Thread` 类中的 `getContextClassLoader()` 和  `setContextClassLoader(ClassLoader cl)` 方法来获取和设置**线程的上下文类加载器**。

线程上下文类加载器不遵循双亲委派模型，逆向使用类加载器。如图，`Bootstrap` 类加载器加载核心 `rt.jar` 包，但该 `SPI` 依赖的实现类位于 `classpath` 路径下，无法通过 `Bootstrap` 类加载器来直接加载，于是 `Bootstrap` 类加载器委托 `Context` 类加载器来加载该实现类 `jdbc.jar`，然后 `rt.jar` 就可以调用 `jdbc.jar` 中的类了。

![线程上下文类加载器工作流程](http://note.youdao.com/yws/public/resource/4abadcd0262eda3859a001aa3e1fcc28/xmlnote/D2F16C22BA0C420FBE3D8D1D41E86664/20090)

一般来说，获取到的线程上下文类加载器默认情况下可能就是 `AppClassLoader`，该类加载器可以直接通过 `getSystemClassLoader()` 来获取。但代码部署到不同服务器上时，可能用到的类加载器并非 `AppClassLoader`。所以使用 `ContextClassLoader` 总会获取到与当前程序相同的 ClassLoader，从而避免一些问题。



## Reference

[虚拟机类加载机制 深入理解Java虚拟机总结](https://www.jianshu.com/p/20f902788988)

[深入理解Java类加载机制](https://www.jianshu.com/p/8cab58ac37e3)

[java类到底是如何加载并初始化的？](https://www.cnblogs.com/jimxz/p/3974939.html)

[类加载之 <clinit>() 和 <init>()](https://www.jianshu.com/p/7ff65c3040ec)

[深入理解JVM内存分配和常量池](https://www.cnblogs.com/zzuli/p/9403928.html)

[ZhouJ00 Blog-类变量和类方法解析](https://zhouj000.github.io/2019/03/27/java-base-jvm6/)

《深入理解 Java 虚拟机》

[深入理解java类加载器](https://blog.csdn.net/javazejian/article/details/73413292)