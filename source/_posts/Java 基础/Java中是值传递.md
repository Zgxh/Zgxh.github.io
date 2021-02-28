---
title: Java中是值传递
date: 2020-02-21 12:30:43
tags: Java
categories: 开发
---

<center>

# Java中只有值传递
</center>
今天看了一篇博客，对java中的参数传递有了进一步的认识。

> 之前的错误认识：传递的参数如果是普通类型，就是值传递；如果是对象，就是引用传递。

## 值传递与引用传递
> 值传递：调用函数时将实参复制一份传递给函数，如果后续在函数中对这个参数进行了修改，不会影响被复制的实参。
> 引用传递：调用函数时直接把实参的地址传递给函数，如果后续在函数中对这个参数进行了修改，将直接影响到被传递的实参。

假设有一个对象：
```java
class User {

    public int id;

    public String name;

    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }
}
```


之前会迷惑人的例子，乍一看是引用传递：
```java
public class Value {

    public static void main(String[] args) {
        User mainUser = new User(1, "yuyang");
        System.out.println(mainUser);
        Value value = new Value();
        value.change(mainUser);
        System.out.println(mainUser);
        System.out.println(mainUser.id + mainUser.name);
    }

    public void change(User user) {
        user.name = "zgxh";
        System.out.println(user);
    }
}
```
输出：
> test.User@5594a1b5
test.User@5594a1b5
test.User@5594a1b5
1zgxh

然而并不是真的引用传递。下面的例子说明虽然传递了对象，但没改变被传递变量：
```java
public class Value {

    public static void main(String[] args) {
        User mainUser = new User(1, "yuyang");
        System.out.println(mainUser);
        Value value = new Value();
        value.change(mainUser);
        System.out.println(mainUser);
        System.out.println(mainUser.id + mainUser.name);
    }

    public void change(User user) {
        user = new User(2, "zgxh");
        System.out.println(user);
    }
}
```
输出：
> test.User@5594a1b5
test.User@6a5fc7f7
test.User@5594a1b5
1yuyang

**总结：Java中只有值传递，对于对象参数，值的内容是对象引用的地址。**