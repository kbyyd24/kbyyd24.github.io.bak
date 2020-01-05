---
title: 使用AOP获取RequestBody
date: 2016-11-20
updated: 2016-11-20
---

一开始使用`spring拦截器`拦截请求记录日志,对于请求路径、header这些都很好获取,唯独`POST`请求无法获取其中的`RequestBody`.

原因很明显,`RequestBody`是只能读取一次的,如果在拦截器中读取了,就无法通过`@RequestBody`注解去获取,因为这个数据是内存中的流数据.

那要如何做呢?明显就需要用到`aop`了

# 利用`AOP`读取`RequestBody`

`aop`就是用来做切面的,具体概念这里不说了,自行谷歌.这里就讲具体问题的思路.

这里用`aop`去切入`Controller`里面所有`public`的方法,利用`JoinPoint`获取参数,从而得到`RequestBody`.
得到`RequestBody`后使用`Spring Boot`的日志打印

## 创建一个切面

```java
@Aspect
@Component
public class RequestMapAspect {}
```

- `@Aspect`注解表明这是一个切面
- `@Componet`注册为组件,否则无法使用`Spring Boot`的日志

## 声明一个`logger`

```java
private final Log log = LogFactory(this.getClass());
```

- 这里的`Log`与`LogFactory`均在`org.apache.commons.logging`包下,**不是`slf4j`,不是`logback`也不是`log4j`**!

## 声明切点

```java
@Pointcut(value = "excution(public * cn.gaoyuexiang.controller.*.*(..))")
public void controllerLog() {}
```

## 编写`Before`获取参数

```java
@Before("controllerLog()")
public void before(JoinPoint point) {
  logger.info("controller aspect begging");
  Object[] args = point.getArgs();
  for (Object arg : args) {
    logger.info("arg: " + arg); 
  }
  String method = point.getSignature().getDeclaringTypeName() + '.' + point.getSignature().getName();
  logger.info("aspect finishing");
  logger.info("calling " + method);
}
```

- `JoinPoint.getArgs()`可以得到目标方法的参数,返回类型为`Object[]`
- `RequestBody`作为参数时,因为是在调用方法前已经被`Spring`读取并解析了,所以不存在重复读取的问题.调用目标方法时,`Before`的`aop`被执行,拿到传递给目标的参数
