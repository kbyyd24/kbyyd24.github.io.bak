---
title: 「Java 8 函数式编程」读书笔记——lambda表达式
date: 2017-02-05
updated: 2017-02-05
tags:
  - java
  - lambda
  - Java Lambda
  - Java 8
  - 函数式编程
---

本文是「Java 8 函数式编程」第二章的读书笔记。

# `Lambda`引入的变化

`Lambda`表达式，是**一种紧凑的、传递行为的方式**，从编程思想上来讲，就是**代码即数据**。

过去的`Java`中，存在大量的匿名内部类的使用，会新建一个匿名内部类传入调用的方法中。这种传统的方式，会造成冗余的、不易阅读的代码。

于是`Lambda`诞生了。`Lambda`的语法简化了使用匿名内部类时的模板代码，让程序员专注于编写想要执行的行为，也让代码更加简洁易读。

# `Lambda`表达式的形式

```java
Runnable runable = () -> System.out.println("Hello Lambda");//1
runable = () -> {
  System.out.print("Hello");
  System.out.println(" Lambda");
};//2
ActionListener listener = event -> System.out.println("get event");//3
BinaryOperator<Long> add = (x, y) -> x + y;//4
BinaryOperator<Long> minux = (Long x, Long y) -> x - y;//5
```

常见的`Lambda`表达式有以上5种，每个`Lambda`表达式都可以分为三个部分：

- 参数部分：`()` `event` `(x, y)` `(Long x, Long y)`
- 将参数和表达式主体分开的符号：`->`
- 表达式主体

## 参数的形式

`Lambda`表达式可以看作是匿名内部类的简写形式，参数也就是使用匿名内部类时实现的方法的参数。

- 有的方法不需要参数，如`Runnable`的`run`方法，所以使用`()`代表参数部分。
- 有的方法只需要一个参数且类型确定，如`ActionListener.actionPerformed`方法，可以直接使用参数，不需要指定类型，也不需要加括号。
- 有多个参数时，必须要加上括号，把参数扩起来
- 当声明参数类型时，无论有多少个参数，都需要加括号

## 表达式主体的形式

表达式可以只有一行代码，也可以有多行代码；有的表达式有返回值，有的没有。

- 只有一行代码的表达式不需要`{}`
  - 如果有返回值，不用写`return`，表达式会把这行代码的返回值作为返回值
  - 如果使用了`{}`，则需要显式的写出`return`
- 有多行代码的表达式必须使用`{}`
  - 如果有返回值，需要显式的写`return`

# 引用值，而不是变量

匿名内部类中，如果想要引用其所在方法中的变量，需要将其声明为`final`。这意味着你实际引用的是一个值，而不是变量。

在`Java 8 `中，虽然可以引用非`final`的变量，但这个变量必须是**既成事实上的`final`**，如果对变量进行修改，将无法通过编译。这意味着`Lambda`表达式仍然是引用的一个值，而不是变量。

实际上可以通过使用数组来绕开编译器，但是这样做之前应该考虑一下你的代码逻辑是否正确。

# 函数接口

只有一个抽象方法的接口叫做函数接口。

JDK中最重要的函数接口：

|     Interface     | Argument | Return  |       e.g.        |
| :---------------: | :------: | :-----: | :---------------: |
|   Predicate<T>    |    T     | boolean |      fliter       |
|    Consumer<T>    |    T     |  void   |      forEach      |
|  Function<T, R>   |    T     |    R    |        map        |
|    Supplier<T>    |   None   |    T    | factory function  |
| UnaryOperator<T>  |    T     |    T    |   modify String   |
| BinaryOperator<T> |  (T, T)  |    T    | add two instances |

# 类型推断

`Java 8`为新成员`Lambda`表达式提供了类型推断的支持，在不需要声明参数类型的`Lambda`表达式中表现的有为明显。形如：

```java
BinaryOperator<Integer> add = (x, y) -> x + y;
```

的表达式得以通过编译并正确执行，就是因为`JVM`通过泛型参数`Integer`推断出了方法参数的类型。