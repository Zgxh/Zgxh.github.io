# Java 堆中的对象内存

## 1. java中的对象指向问题

~~~java
public class HeapMemory {
    
    private Object obj1 = new Object();

    public static void main(String[] args) {
        Object obj2 = new Object();
    }
}
~~~

### 1.1 方法区指向堆

类的属性存放在 **方法区** 中，该属性对应的实例存放在堆中。即obj1符号引用在方法区中，指向堆中对象实例内存。

<img src="http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/AA13AB640B604C289439818851FEF181/9119" style="zoom:80%;" />

### 1.2 虚拟机栈指向堆

类的方法中的属性存放在虚拟机栈的 **局部变量表** 中，对象实例同样存在在堆中。

<img src="http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/11BC3F03FB5E4A5C9C5AEBADBC7B9A51/9121" alt="zhan" style="zoom:67%;" />

## 2. 堆中的对象实例的布局

### 2.1 对象内存模型

java 中的对象内存布局：

- 对象头 Header
  - mark word ： 8 字节
  - class pointer ： 未指针压缩时 8 字节，压缩后 4 字节
  - 数组 length （数组对象特有）： 4 字节
- 对象实例数据 Instance Data ：
- 对齐填充 Padding ： 填充为 8 字节的整数倍

![a](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/88CB4B6F16FA4A71A606DA0E62EC8BE5/9123)

### 2.2 Object obj = new Object() 占用的字节

实例数据为空，mark word 占 8 字节，其他：

- 若开启了类指针压缩 则 class pointer 占4字节，然后 padding 填充 4 字节
- 若没开启，则 class pointer 占 8 字节，padding 不填充

所以，一共占用 16 字节。

## 3. 堆对象的访问

### 3.1 句柄访问

堆中划分出一块空间来存放 **句柄池** , 即 方法区/栈帧 中的对象引用是存放的句柄地址，通过句柄池再在堆中查找对象的实际地址。

<img src="http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/D6A61A98C473467BAB6B1CA90AE2747B/9125" style="zoom:80%;" />

### 3.2 直接指针访问 (hot spot)

堆中直接存放对象的实例数据，即 方法区/栈帧 中的对象引用直接指向堆实例的地址。

<img src="http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/CC077F51710441DEAFC4D5B430B93D07/9128" alt="" style="zoom:80%;" />