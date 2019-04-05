---
title: 《Java 核心技术》读书笔记
date: 2019-02-10T10:23:12.204Z
tags:
    - Java
category: 编程语言
---

《Jave 核心技术》读书笔记。

<!-- more -->

# 对象与类

## UML

![](Java%20核心技术/2019-02-10-19-34-30.png)

## 访问器方法

修改对象的方法称为**更改器方法（*mutator method*）**。相反，只访问对象而不修改对象的方法称为**访问器方法（*accessor method*）**。

**警告：**注意不要编写返回引用可变对象的访问器方法。

## 静态方法

两种情况下使用静态方法：
* 一个方法不需要访问对象状态，其所需参数都是通过显式参数提供（例如：`Math.pow`）。
* 一个方法只需要访问类的静态域（例如：`Employee.getNextId`）。

静态方法还有另外一种常见的用途。类似 `LocalDate` 和 `NumberFormat` 的类使用静态工厂方法（*factory method*）来构造对象。

## 方法参数

Java 程序设计语言总是采用**按值调用**。

## 类设计技巧

1. 一定要保证数据私有
2. 一定要对数据初始化
3. 不要在类中使用过多的基本类型
    用其他的类代替多个相关的基本类型的使用。这样会使类更加易于理解且易于修改。
4. 不是所有的域都需要独立的域访问器和域更改器。
5. 将职责过多的类进行拆分
6. 类名和方法名要能够提现他的职责
    命名类名的良好习惯是采用一个名词、前面有形容词修饰的名词或动名词修饰名词。
7. 优先使用不可变的类
    要尽可能让类是不可变的，这是一个很好的想法。

# 继承

## Object：所有类的超类

在 Java 中，只有**基本类型（*primitive types*）**不是对象，例如，数值、字符和布尔类型的值都不是对象。

所有的数组类型，不管是对象数据还是基本类型的数组都是扩展了 Object 类。

## 对象包装器与自动装箱

有时，需要将 `int` 这样的基本类型转换为对象。所有的基本类型都有一个与之对应的的类。

**`ArrayList<Integer>` 的效率远远低于 `int[]` 数组。**

![](Java%20核心技术/2019-02-10-20-03-39.png)

## 继承的设计技巧

1. 不要使用受保护的域
2. 除非所有的继承方法都有意义，否则不要使用继承
3. 在覆盖方法时，不要改变预期的行为
4. 不要过多的使用反射

# 异常、断言和日志

![](Java%20核心技术/2019-02-10-20-45-43.png)

由程序错误导致的异常属于 `RuntimeException`；而程序本身没有问题，但由于像 I/O 错误这类问题导致的异常属于其他异常。

![](Java%20核心技术/2019-02-10-20-48-00.png)

**警告：**如果在子类中覆盖了超类的一个方法，子类方法中声明的受查异常不能比超类方法中声明的异常更通用（也就是说，子类方法中可以抛出更特定的异常，或者根本不抛出任何异常）。

*在 C++ 中 如果没有给出 throw 说明，函数可能会抛出任何异常。而在 Java 中，没有 throw 说明符的方法将不能抛出任何受查异常。*

## 捕获异常

**应该捕获那些知道如何处理的异常，而将那些不知道怎样处理的异常继续进行传递。**

### 再次抛出异常

在 cache 子句中可以抛出一个异常，这样做的目的是改变异常的类型。

不过，可以有更好的处理方法：

``` Java
try
{
    access the database
}
catch (SQLException e)
{
    Throwable se = new ServletException("database error");
    se.initCause(e);
    throw se;
}
```

当捕获到异常时，就可以使用下面这条语句重新得到原始异常:

``` Java
Throwable e = se.getCauseO;
```

强烈建议使用这种包装技术。这样可以让用户抛出子系统中的高级异常，而不会丢失原始异常的细节。

**提示: **如果在一个方法中发生了一个受查异常，而不允许抛出它，那么包装技术就十分 有用。我们可以捕获这个受查异常，并将它包装成一个运行时异常。

### finally 子句

不管是否有异常被捕获，finally 子句中的代码都被执行。

强烈建议解搞合 try/catch 和 try/finally 语句块。这样可以提高代码的清晰度。例如:

``` Java
InputStrean in = . . .; 
try
{
    try 
    {
        code that might throw exceptions 
    }
    finally
    {
        in.close(); 
    }
}
catch (IOException e) 
{
    show error message 
}
```

内层的 try 语句块只有一个职责，就是确保关闭输入流。外层的 try 语句块也只有一个职责，就是确保报告出现的错误。这种设计方式不仅清楚，而且还具有一个功能，就是将会报告 finally 子句中出现的错误。

### 带资源的 try 语句

带资源的 try 语句(*try-with-resources*) 的最简形式为：

``` Java
try (Resource res = . . .) 
{
    work with res 
}
```

try 块退出时，会自动调用 `res.close()`。

如果 try 块抛出一个异常，而且 close 方法也抛出一个异常，这就会带来一个难题。带资源的 try 语句可以很好地处理这种情况。原来的异常会重新抛出，而 `close` 方法抛出的异常会被抑制。这些异常将自动捕获，并由 `addSuppressed` 方法增加到原来的异常。如果对这些异常感兴趣，可以调用 `getSuppressed` 方法，它会得到从 `close` 方法抛出并被抑制的异常列表。

## 使用异常机制的技巧

使用异常的基本原则：只在异常的情况下使用异常机制。

1. 利用异常的层次结构
2. 在检查错误时，“苛刻”要比放任更好
3. 不要羞于传递异常

## 调试技巧

1. 一个不太为人所知但却非常有效的技巧是在每一个类中放置一个单独的 main 方法。这样就可以对每一个类进行单元测试。
2. 要想观察类的加载过程，可以用 `-verbose` 标志启动 Java 虚拟机。这样就可以看到如下所示的输出结果:
    ![](Java%20核心技术/2019-02-10-21-13-46.png)
    有时候，这种方法有助于诊断由于类路径引发的问题。