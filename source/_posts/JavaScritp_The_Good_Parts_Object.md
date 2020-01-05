---
title: 「JavaScript 语言精粹」读书笔记--对象
date: 2017-03-04
updated: 2017-03-04
---

前两章介绍基础, 没什么笔记好记录. 这是第三章.

## 什么是对象

在`JavaScript`中, 除了简单数据类型(数字, 字符串, 布尔值, `null`和`undefined`), 其他所有的值都是对象`Object`.

其中`number` `string`和`boolean`虽然拥有方法, 但他们并不是`object`, 因为他们是**不可变的**.

`JavaScript`中的对象是**可变的** **键控集合**. 在`JavaScript`中, 数组/函数/正则表达式/对象本身都是对象.

对象是属性的容器. 属性由`K/V`组成, 属性名可以是包括空字符串在内的任意字符串, 属性值可以是**除`undefined`之外**的任何值. 这意味着**对象可以包含对象**.

对象是无类型(`class-free`)的, 对新属性的键值没有限制.

`JavaScript`中包含一种**原型链**的特征, 允许对象继承另一个对象的属性. 这一特性可以用来减少时间和内存消耗.

## 对象字面量

### 标识符

`JavaScript`中的标识符由字母开头, 可由字母, 数字和下划线组成, 但是不能使用保留字, 如: `abstract` `boolean` `if`等.

---

一个对象的字面量就是包围在一对花括号中的零个或多个`K/V`对.

```javascript
var empty = {};
var person = {
  "first-name": "melo",
  "last-name": "Gao"
};
```

如果属性名是合法的标识符, 且不是保留字, 则不强制要求使用引号. 如:

```javascript
var auther = {
  firstName: "Douglas",
  familyName: "Crockford"
}
```

## 检索

有两种方式检索到对象的属性:

```javascript
person['first-name'] //1 melo
auther.firstName //2 Douglas
empty.first_name //3 undefined
```

作者推荐优先使用方式`2`和`3`, 理由是可读性更好. 但方式`1`可以让我们通过修改参数值而达到动态访问的目的, 如:

```javascript
key = 'first-name';
person[key] //melo
key = 'firstName';
auther[key] //Douglas
```

另外, 对于第三种情况, 要注意判断返回的值, `undefined`会被判断为`false`. 如果不做判断, 会抛出`TypeError`异常.

## 更新

可直接对对象中的属性赋值, 就像对`Java`中的`public`属性赋值一样. 当属性不存在时, 该属性会扩充到对象中.

## 引用

`JavaScript`通过引用传递对象, 他们永远不会被复制.

## 原型

每个对象都连接到一个原型对象, 并且可以从中继承属性. 所有**通过对象字面量创建**的对象都连接到`Object.prototype`, 它是`JavaScript`中的标配对象. (这有点像`Java`中所有类都是`Object`的子类.)

原型在更新属性时是不起作用的, 如果对象没有相应属性, 会扩充该属性.

只有在检索属性时, 原型才可能起作用. 如果在对象中没有找到目标属性, 则会在它的原型对象中查找. 如果原型对象中还是没有找到, 再到它的原型对象中查找, 依此类推, 直到找到该属性, 或者在`Object.prototype`中也找不到为止. 如果该属性不在此原型链中, 则得到`undefined`. 这个过程叫做**委托**.

## 反射

使用关键字`typeof`可以查看任何值的类型.

```javascript
typeof person //'object'
typeof 2333 //'number'
typeof '2333' //'string'
typeof true //'boolean'
typeof Object //'function'
typeof auther.firstName //'string'
```

原型链中的任何属性都会产生值.

另一个方法是`hasOwnProperty`, 用于判断对象是否拥有某个属性, 但不会查找原型链.

```javascript
auther.hasOwnProperty('firstName')
```

## 枚举

使用`for in`循环可以枚举对象中的所有属性. 但是这种枚举是无序的, 而且会遍历整个原型链, 所以需要做判断.

```javascript
var name;
for (name in auther) {
  if (auther.hasOwnProperty(name)) {
    //do something
  }
}
```

## 删除

`delete`运算符用于删除对象的属性, 但不会触及原型链中的任何对象.

```javascript
delete auther.familyName
```

## 减少全局变量污染

无论何时, 使用大量全局变量都不是一个值得推崇的做法. 我们可以定义一个全局的对象, 把需要的全局变量纳入其名称空间, 降低模块间的冲突.
