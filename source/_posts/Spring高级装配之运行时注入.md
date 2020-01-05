---
title: Spring高级装配之运行时注入
date: 2016-08-17
updated: 2016-08-17
---

* 
{:toc #zhuru}

运行时注入与硬编码注入是相对的。硬编码注入在编译时就已经确定了，运行时注入则可能需要一些外部的参数来解决。

`Spring`提供的两种在运行时求值的方式：

* 属性占位符(Property placeholder)
* `Spring`表达式语言(SpEL)

## 注入外部的值

使用`@PropertySource`注解可以引入`.properties`文件，使用其中的值。

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
public class JDBCConfig {
    @Autowired
    Environment env;
    
    @Bean
    public DataSource dataSource() {
        env.getProperties("driver");
        ...
    }
}
```

### 深入了解`Spring`中的`Environment`

上例的`Environment`有如下方法获取属性

* String getProperty(String key);
* String getProperty(String key, String defaultValue);
* T getProperty(String key, Class<T> type);
* T getProperty(String key, Class<T> type, T defaultValue);

这几个重载方法的作用顾名思义。其中第一、三个方法获取一个不存在的属性时，会抛出`IllegalStateException`异常。

可以使用`containsProperty(String key)`方法查看是否存在某个属性。

其他相关方法：

* `Class<T> getPropertyAsClass(String key, Class<T> targetType)` : 将获取的属性转换为类
* `String[] getActiveProfiles()` : 返回激活`profile`名称的数组
* `String[] getDefaultProfiles()` : 返回默认`profile`名称的数组
* `boolean acceptsProfiles(String... profiles)` : 如果`environment`支持给定的`profile`，则返回`true`

### 解析属性占位符

使用占位符，可将属性定义到外部的`.properties`文件中，然后使用占位符插入到`bean`中。占位符使用`${...}`包装属性名称。

在`Java`配置中使用`@Value`注解。

```java
public BlankDisc(@Value("${disc.title}") String title,
            @Value("${disc.artist}") String artist) {
    this.title = title;
    this.artist = artist;
}
```

使用占位符必须配置一个`PropertySourcesPlaceholderConfigurer bean`，它能够基于`Spring Environment`及其属性来解析占位符。

```java
@Bean
public PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
    return new PropertySourcesPlaceholderConfigurer();
}
```

## 使用`Spring`表达式语言进行装配

`SpEL`主要特性：

* 使用`bean`的`ID`来引用`bean`
* 访问对象的属性和方法
* 可对值进行算数、关系和逻辑运算
* 正则表达式匹配
* 集合操作

`SpEL`还可以用在`DI`之外的地方

### `SpEL`样例

`SpEL`表达式要放在`#{ ... }`中，里面的"..."就是`SpEL`表达式。

* #{1}

常量，结果始终为`1`

* #{T(System).currentTimeMillis()}

`T()`表达式会将`java.lang.System`视为`Java`中对应的类型，然后调用其方法，获取当前时间戳。

* #{dataSource.user}

`dataSource`为声明的其他`bean`，这里可以获取它的属性`user`

* #{systemProperties['username']}

通过`systemProperties`对象获取系统属性

### 表示字面量

可表示的字面量有`int`,`float/double`,`String`,`boolean`，其中浮点值可以用科学技术法表示：`#{6.18E3}`

### 引用`bean`、属性和方法

| 引用对象 | 表达式 |
|:-:|:-:|
| bean | `#{dataSource}` |
| bean's field | `#{dataSource.user}` |
| bean's method | `#{dataSource.getPassword()}` |
| bean's method's method | `#{dataSource.getPassword().toUpperCase()}` |

如果方法返回值为`null`，第四种情况会抛出`NullPoniterException`。可以使用：

`#{dataSource.getPassword()?.toUpperCase()}`

其中的`?.`运算符能够在访问前确保不为`null`，否则返回`null`。

### 在表达式中使用类型

使用`T()`表达式来访问`Java`类中的`static`方法和常量，在括号内的是类名，返回一个`Class`对象，然后调用其方法和常量。

### `SpEL`运算符

| 运算符类型 | 运算符 |
|:-:|:-:|
| 算数运算 | +, -, *, /, %, ^ |
| 比较运算 | <, >, ==, <=, >=, lt, gt, eq, le, ge |
| 逻辑运算 | and, or, not, \| |
| 条件运算 | ?: (ternary), ?: (Elvis) |
| 正则表达式 | matches |

* `Elvis`运算符
 
利用三元运算符来检查场景：`#{disc.title ?: 'Rattle and Hum'}`，当`disc.title`为`null`时，返回`"Rattle and Hum"`。
 
名称的来历据说是因为'?'长得像猫王的头发。。。 :astonished::astonished::astonished:
 
* 正则表达式

正则表达式利用`matches`来支持正则匹配。

### 计算集合

* 引入一个元素 : `#{jukebox.songs[4].title}`
* 随机选取 : `#{jukebox.songs[T(Math).random() * jukebox.songs.size()].title}`
* 从`String`中获得`char` : `#{'This is a test'[2]}`
* 使用`.?[]`进行过滤，得到符合条件的子集 : `#{jukebox.songs.?[artist eq 'Aerosmith']}`
* 使用`.^[]`和`.$[]`进行过滤，得到第一个和最后一个匹配项
* 使用`.![]`从集合的每个成员选择特定属性放入新集合中 : `#{jukebox.songs.![title]}`

> 最后四个表达式有点像`lambda`表达式

`SpEL`的表达式可以相互组合使用。

> 更多`Spring`学习笔记：[https://github.com/kbyyd24/spring.demo.test/issues](https://github.com/kbyyd24/spring.demo.test/issues)