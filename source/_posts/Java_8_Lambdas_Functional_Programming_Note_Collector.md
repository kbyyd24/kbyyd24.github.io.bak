---
title: 「Java 8 函数式编程」读书笔记——高级集合类和收集器
date: 2017-02-10
updated: 2017-02-10
tags:
  - java
  - lambda
  - Java Lambda
  - Java 8
  - 函数式编程
---
本章是该书的第五章, 主要讲了方法引用和收集器

# 方法引用

形如:

```java
artist -> artist.getName()
(String arg) -> arg.length()
```

这样的表达式, 可以简写为:

```java
Artist::getName
String::length
```

这种简写的语法被称为`方法引用`. 方法引用无需考虑参数, 因为一个方法引用可以在不同的情况下解析为不同的`Lambda`表达式, 这依赖于`JVM`的推断.

## 方法引用的类型

方法引用可以分为四类:

- 引用静态方法: `ClassName::staticMethodName`, 比如: `String.valueOf`
- 引用特定实例方法: `object::instanceMethodName`, 比如: str::toString
- 引用特定类型的任意对象的实例方法: `ClassName::instanceMethodName`, 比如: `String::length`
- 引用构造方法: `ClassName::new`, 比如: `String::new`

# 元素顺序

当我们对集合进行操作时, 有时希望是按照一定的顺序来操作, 而有时又希望是乱序的操作. 有两个方法可以帮助我们进行顺序的操作.

## 乱序

`BaseStream.unordered()`方法可以打乱顺序, 科技将本来有序的集合变成无序的集合

## 排序

`Stream.sorted`方法有两个签名, 一个无参, 一个有参数`Comparator<? super T> comparator`

- 无参的方法要求`T`实现了`Comparable`接口
- 有参方法需要提供一个比较器

# 收集器

收集器是一种通用的, 从流中生成复杂值的结构. 将其传给`collect`方法, 所有的流就都可以使用它. 而下面提到的单个收集器, 都可以使用`reduce`方法模拟.

## 转换成集合

我们可以使用`Collectors`中的静态方法`toList()` `toSet()`等, 将流收集为`List`或`Set`

```java
stream.collect(toList())
stream.collect(toSet())
```

我们不需要关心具体使用的是哪一种具体的实现, `Stream`类库会为我们选择. 因为我们可以利用`Stream`进行并行数据处理, 所以选择是否线程安全的集合十分重要.

当然我们也可以指定使用哪一种实现来进行收集:

```java
stream.collect(toCollection(ArrayList::new))
```

## 转换成值

`Collectors`类提供了很多的方法用于转化值, 比如`counting` `maxBy` `minBy`等等, 可以查看`javadoc`了解.

目前了解到的是, 这三个方法都可以使用`Stream`中的`count` `max` `min`方法代替, 而不需要作为`collect`方法的参数

## 数据分割

有时我们想按照一个条件把数据分成两个部分, 而不是只获取符合条件的部分, 这时可以使用`partitioningBy`方法收集. 将它传入`collect`方法, 可以得到一个`Map<Boolean, List>`, 然后就可以对相应的数据进行处理了.

## 数据分组

`groupingBy`方法可以将流分成多个`List`, 而不仅仅是两个, 接收一个`Lambda`表达式作为参数, 其返回值作为`key`, 最后的结果也是一个`Map`, 形如`Map<String, List>`. 这一方法类似于`SQL`中的`group by`

## 生成字符串

如果要从流中得到字符串, 可以在得到`Stream<String>`之后使用`Collectors.joining`方法收集. 该方法接收3个`String`参数, 分别是`分隔符` `前缀` `后缀`

```java
artists.stream()
  .map(Artist::getName)
  .collect(Collectors.joining(",", "[", "]"));
```

## 组合收集器

我们可以将收集器组合起来, 达到更强的功能. 书上举了两个栗子:chestnut:

- 例一

```java
public Map<Artist, Long> numberOfAlbums(Stream<Album> albums) {
  return albums
    .collect(
    groupingBy(Album::getMainMusicina, counting()));
}
```

这个方法的目的是统计每个歌手的作品数目. 如果不组合收集器, 我们先用`groupingBy`得到一个`Map<Artist, List<Album>>`之后, 还要去遍历`Map`得到统计数目, 增加了代码量和性能开销.

上面的`counting`方法类似于`count`方法, 作用于`List<Album>`的流上.

- 例二

```java
public Map<Artist, List<String>> nameOfAlbums(Stream<Album> albums) {
  return albums
    .collect(
    groupingBy(Album::getMainMusician,
              mapping(Album::getName, toList())));
}
```

这个方法的目的是得到每个歌手的作品名称列表. 如果不组合收集器, 我们将会先得到一个`Map<Artist, List<Album>>`. 然而, 我们只想得到作品名称, 也就是一个`List<String>`, 组合`mapping`收集器可以帮助我们实现效果.

`mapping`收集器的功能类似于`map`, 将一种类型的流转换成另一种类型. 所以类似的, `mapping`并不知道要把结果收集成什么数据结构, 它的第二个参数就会接收一个普通的收集器, 比如这里的`toList`, 来完成收集.



这里的`counting`和`mapping`是我们用到的第二个收集器, 用于收集最终结果的一个子集, 这些收集器叫做`下游收集器`. 

## 定制收集器

定制收集器看起来麻烦, 其实抓住要点就行了.

### 使用`reduce`方法

前面说过, 这些收集器都可以使用`reduce`方法实现, 我们定制收集器, 实际上就是为`reduce`方法编写三个参数, 分别是:

- identity
- accumulator
- combiner

关于这三个参数的意义, 如果不太理解, 可以看看这个答案: [https://segmentfault.com/q/1010000004944450](https://segmentfault.com/q/1010000004944450)

我们可以设计一个类, 为这三个参数设计三个方法, 再提供一个方法用于获取目标类型(如果这个类就是目标类型的话, 可以不提供这个方法)

###  实现`Collector`接口

如果不想显式的使用`reduce`方法, 我们只需要提供一个类, 实现`Collector`接口.

该接口需要三个泛型参数, 依次是:

- 待收集元素的类型
- 累加器的类型
- 最终结果的类型

需要实现的方法有:

- `supplier`: 生成初始容器


- `accumulator`: 累加计算方法
- `combiner`: 在并发流中合并容器
- `finisher`: 将容器转换成最终值
- `characteristics`: 获取特征集合

多数情况下, 我们的容器器和我们的目标类型并不一致, 这时, 需要实现`finisher`方法将容器转化为目标类型, 比如调用容器的`toString`方法.

有时我们的目标类型就是我们的容器, `finisher`方法就不需要对容器做任何操作, 而是通过设置`characteristics`为`IDENTITY_FINISH`, 使用框架提供的优化得到结果.

详细讲解可以参见[http://irusist.github.io/2016/01/04/Java-8%E4%B9%8BCollector/](http://irusist.github.io/2016/01/04/Java-8%E4%B9%8BCollector/)

# `Map`新增方法

`Java 8`为`Map`新增了很多方法, 可以通过搜索引擎轻松找到相关文章. 这里举几个书中提到的相关方法.

- `V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)`
- `V computeIfPresent(K key, BiFunction<? super K, ? super V, extends V> remappingFunction)`
- `V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)`

这三个方法类似, 都是根据`key`来处理, 只是`Lambda`表达式的执行条件不同, 从函数名就可以看出来. 不过要注意`Lambda`表达式的参数, 第一个方法的`Lambda`只需要一个参数`key`, 后面两个方法的`Lambda`需要两个参数`key`和`value`, 而`compute`方法的`Lambda`中的`value`参数可能为`null`.

- `V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction)`

此方法用于合并`value`, 新`value`在第二个参数给出. `Lambda`表达式规定合并方法, 其两个参数依次是`oldValue`和`newValue`, `oldValue`是原`Map`的`value`, 可能为空; `newValue`为`merge`方法的第二个参数.

- `void forEach(BiConsumer<? super K, ? super V> action)`

通过`forEach`方法, 不再需要使用外部迭代来遍历`Map`.