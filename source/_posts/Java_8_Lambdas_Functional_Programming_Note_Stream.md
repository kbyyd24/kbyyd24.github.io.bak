---
title: 「Java 8 函数式编程」读书笔记——流
date: 2017-02-06
updated: 2017-02-06
tags:
  - java
  - lambda
  - Java Lambda
  - Java 8
  - 函数式编程
---

本文是「Java 8 函数式编程」第三章的读书笔记，章名为流。本章主要介绍了外部迭代与内部迭代以及常用的高阶函数。

# 外部迭代与内部迭代

## 外部迭代

过去我们要对一个`List`进行迭代时，往往会采用如下方式：

```java
int count = 0;
for (Artist artist : artists) {
  if (artist.isFrom("London")) {
    count++;
  }
}
```

而这种方法的原理，其实是先调用`iterator`方法，然后再迭代，等效于如下代码：

```java
int count = 0;
Iterator<Artist> iterator = artists.iterator();
while (iterator.hasNext()) {
  Artist artist = iterator.next();
  if (artist.isFrom("London")) {
    count++;
  }
}
```

这样的迭代方式，把迭代的控制权交给了`iterator`对象，让其控制整个迭代过程，这就叫做`外部迭代`。

外部迭代需要我们自己编写迭代的控制代码，显得十分繁琐。特别是对于`Map`对象，繁琐到我都不想给出例子。

外部迭代将行为和方法混为一谈，难以对代码进行重构操作。

## 内部迭代

与之相对的就是内部迭代了。**内部迭代就是把迭代的控制权交给了集合本身，让集合自己实现相应的迭代，而调用者并不需要关心如何迭代**。

要使用内部迭代，需要使用`Java 8`中新增的接口`Stream`。而集合框架都已经包含了一个`stream()`方法，用于获得`Stream`对象。

```java
long count = artists.stream()
  .filter(artist -> artist.isFrom("London"))
  .count();
```

这个例子就是使用的内部迭代。先获取`stream`对象，然后调用`filter`方法过滤，最后统计符合条件的个数。

# 实现机制

在`Java`中调用一个方法，通常会立即执行操作。然而`Stream`里的一些方法却不太一样，它们返回的对象不是新的集合，而是创建新集合的配方。我们通过一个例子说明：

```java
Stream<String> names = Stream.of("Bryant", "Jordon", "James")
  .filter(name -> {
    System.out.println(name);
    return name.length() == 6;
  });
System.out.println("counting");
System.out.println(names.count());
```

最终会得到如下输出：

```
counting
Bryant
Jordon
James
2
```

出现这样的结果，原因是

- 像`filter`这样的方法，只会描述`Stream`，最终不会产生新集合的方法叫做`惰性求值方法`
- 像`count`这样会从`Stream`中产生值或集合等结果的方法叫做`及早求值方法`

判断一个操作是惰性求值还是及早求值，只需要看它的返回值

- 如果返回值是`Stream`，则是惰性求值
- 返回的是一个值或`null`，则是及早求值

在对集合使用流操作时，使用惰性求值方法形成一个惰性求值的链，最后用及早求值方法得到结果，而集合只需要迭代一次。

# 常用流操作

- collect：及早求值，常用于生成`List` `Map`或其他复杂的数据结构
- map：惰性求值，将一种类型的数据转换成另一种类型，将一个流中的值转化成一个新的流，类似于`Hadoop`里的`map`
- filter：惰性求值，过滤不符合条件的元素
- flatMap：惰性求值，类似于`map`，只是`Function`参数的返回值限定为`Stream`，用于连接多个`Stream`成为一个`Stream`
- max & min：及早求值，`reduce`方法的特例，返回`Optional`（第四章介绍）对象
- reduce：及早求值，从一组值中生成一个值，类似于`Hadoop`中的`reduce`

# 高阶函数

高阶函数 :point_right: 接收一个函数作为参数，或者返回一个函数的函数。

# 正确使用`Lambda`表达式

- 明确要达成**什么转化**，而不是说明**如何转化**
- 没有副作用：
  - 只通过函数的返回值就能充分理解函数的全部作用
  - 函数不会修改程序或外界的状态
  - 获取值而不是变量（避免使用数组逃过`JVM`的追杀，应该考虑优化逻辑）

