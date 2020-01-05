---
title: 定制Java中的Validation
date: 2017-10-25
updated: 2017-10-25
---
 
# `Java Validation`简介
 
`javax.validation`包提供了大量用于验证数据的`API`工具，可以帮助开发者方便的检验程序的输入输出。最近刚刚接触到这方的知识，还只知道使用其提供的注解工具。

想要使用这个工具，需要导入这个包。以`gradle`为例：

```groovy
compile group: 'javax.validation', name: 'validation-api', version: '2.0.0.Final'
```

但这个包仅仅定义了规范的接口、注解等，并没有提供具体实现。常用的实现是由`Hibernate`提供的：

```groovy
compile group: 'org.hibernate.validator', name: 'hibernate-validator', version: '6.0.3.Final'
compile group: 'org.hibernate', name: 'hibernate-validator-annotation-processor', version: '6.0.3.Final'
```

## 注解使用示例

我们以`@NotEmpty`注解为例，这是由`Hibernate`提供的注解，而不是标准规范注解。

假设我们有一个`Controller`，需要接收一个`POST`请求，`RequestBody`为`JSON`，包含`username`和`password`两个字段。

首先定义我们的`Body`：

```java
@Data
public class LoginCommand {
  @NotEmpty(message = "username cannot be empty")
  private String username;
  @NotEmpty(message = "password cannot be empty")
  private String password;
}
```

- `@Data`注解由`lombok`提供，可以帮助我们创建出一个`Java Bean`。
- `@NotEmpty`注解会在`LoginCommand`对象创建后验证被注解修饰的`field`的值，当`username`为`null`或`""`时，会得到`false`，解析这个注解的`Validator`就会抛出异常，异常信息的`message`就是这里设置的值，最终`Spring`会把这样的异常以`400`响应返回给客户端。

接下来是我们的`Controller`：

```java
@RestController
public class UserController {
  @PostMapping("login")
  public String (@Valid @RequestBody LoginCommand command) {
    return "23333";
  }
}
```

- `@Valid`注解由`javax.validation`提供，一般使用在除基本类型和`String`之外的其他类型的对象上，以允许实现`Validator`的类能够对该对象进行递归的调用。

这样，当我们请求`/login`接口的时候，如果`username`或`password`为`null`或`""`，则会得到`400`的`HTTP`响应。

如果希望系统学习，请移步：[http://hibernate.org/validator/documentation/](http://hibernate.org/validator/documentation/)

# 面临需求

现在我们面临一个需求，对于`POST`的数据，其中一个字段可以为`""`或者是正常的字符串，而不能是`null`或`"   "`。面对这样的情况，我们没法在`Hibernate`提供的库里找到需要的注解。这个时候，为了满足需求，我们需要定制我们的`Validator`以满足需求。

# 通过注解实现定制

解决方法来自：[https://stackoverflow.com/a/43716689/6487869](https://stackoverflow.com/a/43716689/6487869)

最简单的方式，当然是不要写任何实现代码啦。仔细研究`javax.validation`和`Hibernate`提供的注解，我们找到了`@Null`、`@NotEmpty`和`@NotBlank`三个注解。发现组合这几个注解可以帮助我们构造出我们需要的注解。

```
EmptyOrNotBlank = NotBlank | Empty
Empty = !(Null | NotEmpty)
```

结合前面`stack overflow`的回答，我们能够很容易的利用已有的注解构造出`@Empty`和`@NotBlank`这两个注解，并且不需要通过实现接口来完成逻辑代码。

`@Empty`注解

```java
@ConstraintComposition(CompositionType.ALL_FALSE)
@Null
@NotEmpty
@ReportAsSingleViolation
@Target({FIELD, ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {})
public @interface Empty {
    String message() default "";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

`@EmptyOrNotBlank`注解

```java
@ConstraintComposition(CompositionType.OR)
@Empty
@NotBlank
@ReportAsSingleViolation
@Target({ANNOTATION_TYPE, FIELD, METHOD, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = {})
public @interface EmptyOrNotBlank {
    String message() default "";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

# 自定义`Validator`实现定制

解决方法来自：[https://docs.jboss.org/hibernate/validator/5.0/reference/en-US/html/validator-customconstraints.html#validator-customconstraints-validator](https://docs.jboss.org/hibernate/validator/5.0/reference/en-US/html/validator-customconstraints.html#validator-customconstraints-validator)

另一种实现方式当然就是通过实现接口来做到啦，现在我们不用再去组合已有的注解，而是需要为`@Constraint`注解的`validateBy`指定用于验证的类。

于是我们的`@EmptyOrNotBlank`可以写成：

```java
@ReportAsSingleViolation
@Target({ANNOTATION_TYPE, FIELD, METHOD, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = EmptyOrNotBlankValidator.class)
public @interface EmptyOrNotBlank {
    String message() default "";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

然后，我们需要编写`EmptyOrNotBlankValidator`这个类。

```java
public class EmptyOrNotBlankValidator implements ConstraintValidator<EmptyOrNotBlank, String> {

  @Override
  public void initalize(EmptyOrNotBlank constraintAnnotation) {}

  @Override
  public boolean isValid(String object, ConstraintValidatorContext constraintContext) {
    if (object == null) {
      return false;
    }
    if (object.isEmpty()) {
      return true;
    }
    return !object.trim().isEmpty();
  }
}
```

完成之后，就能达到和使用组合注解的方法一样的效果了。对于一些复杂的、组合注解难以实现的验证，更加推荐实现`ConstraintValidator`的方式。
