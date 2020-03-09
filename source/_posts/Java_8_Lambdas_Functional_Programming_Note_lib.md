---
title: 「Java 8 函数式编程」读书笔记——类库
date: 2017-02-08
updated: 2017-02-08
tags:
  - java
  - lambda
  - Java Lambda
  - Java 8
  - 函数式编程
---
本书第四章的读书笔记, 本章主要阐述: 如何使用`Lambda`表达式. 

## 基本类型

考虑到装箱类型过于占用内存, `JDK`提供了针对基本类型的操作, 以达到优化的效果, 如`mapToLong`方法.

对基本类型做特殊处理的方法在命名上有明确规范:

- 如果返回类型为基本类型, 则在基本类型名称前面加`To`
- 如果参数类型是基本类型, 则不加前缀只需类型名即可
- 如果敢接函数使用基本类型, 则在操作名后加`To`, 再加基本类型名, 如`mapToLong`

### summaryStatistics方法

这些为基本类型进行优化的`Stream`还有一些额外的方法, 避免重复实现一些通用方法, 比如`summaryStatistics`方法.

```java
public static void printSummary(List<Production> productions) {
  DoubleSummaryStatistics priceStats = productions.stream()
    .mapToDouble(prod -> prod.getPrice())
    .summaryStatistics();
  System.out.printf("max: %f, min: %f, ave: %f, sum: %f",
                   priceStats.getMax(),
                   priceStats.getMin(),
                   priceStats.getAberage(),
                   priceStats.getSum());
}
```

## 重载解析

`Lambda`表达式作为参数时, 其类型由它的`目标类型`(方法的参数类型)推导得出, 推导过程遵循如下规则:

- 如果只有一个可能的目标类型, 由相应函数接口里的参数类型推导得出
- 如果有多个可能的目标类型, 由最具体的类型推导得出
- 如果有多个可能的目标类型且最具体的类型不明确, 则需要人为指定类型

## @FunctionalInterface

该注解会强制`javac`检查一个接口是否符合函数接口的标准. 如果该注解被添加给一个枚举类型, 类或者另一个注解, 或者接口包含不止一个抽象方法, javac就会报错.

## 默认方法

### 产生原因

由于集合框架的基本接口如`Collection` `Map`等都新增了`stream`方法, 在以前的版本中, 第三方的类库如果实现了`Collection`这样的接口, 必须新增`stream`方法的实现, 否则无法通过`Java 8`的编译. 

为了避免这种情况, `Java 8`中添加的新的语言特性: `默认方法`

### 编写方法

`Java 8`中的任何接口都可以添加默认方法, 使用`default`关键字修饰, 比如`forEach`方法:

```java
default void forEach(Consumer<? super T> action) {
  for (T t : this) {
    action.accept(t);
  }
}
```

因为接口没有成员变量, 所以默认方法只能通过调用子类的方法来修改子类本身.

### 默认方法的重写

假设:

- 接口`A`有默认方法`a`,
- 接口`B`扩展了接口`A`, 并重写了方法`a`
- 类`C`实现接口`A`, 并重写方法`a`
- 类`D`实现接口`B`, 并重写方法`a`

#### 没有重写的情况

1. 一个类实现接口`A`, 则会调用接口`A`的实现
2. 一个类实现接口`B`, 则会调用接口`B`的实现
3. 继承于`C` `D`的类, 无论是否实现了接口`A`或`B`, 都将会调用`C` `D`的实现
4. 实现`A`与`B`, 但没有继承`C`或`D`的类将无法通过编译

#### 有重写的情况

无论继承情况如何, 只要重写了默认方法, 都将调用自己的实现

#### 三定律

1. 类胜于接口
2. 子类胜于父类
3. 没有规则三, 如果上面两条不适用, 子类需要实现该方法, 或声明为抽象方法

## 接口的静态方法

如果一个方法有充分的语义原因和某个概念相关, 那么就应该将方法和相关的类或接口放在一起, 而不是放到另一个工具类中. 基于这个原因, `Java 8`提供了接口的静态方法的支持. `Stream`接口中就包含多个静态方法用于生成`Stream`对象.

## Optional

`Optional`是为核心类库新设计的一个数据类型, 用于替换`null`值. 它可以接收一个泛型参数.

- 调用`get`方法获得泛型类型的对象.
- `isPresent`方法判断是否为空
- `orElse` `orElseGet` `orElseThrow`方法可以自由定制为空时的返回值/抛出异常