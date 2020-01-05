---
title: 「JavaScript 语言精粹」读书笔记--函数
date: 2017-03-06
updated: 2017-03-06
---

## 函数对象

`JavaScript`中函数就是对象. 函数对象连接到`Function.prototype`.

当把一个函数当作构造函数(使用`new`关键字)使用时, 新创建的对象的原型就是该函数的`prototype`对象. 我们可以通过给`prototype`设置属性而达到让该类对象拥有同样的公共属性的目的.

新创建的对象有一个`__proto__`属性, 指向该函数的`prototype`对象.

## 函数字面量

函数对象通过函数字面量来创建. 函数字面量可以出现在任何允许表达式出现的地方, 甚至可以被定义在函数内部.

内部函数处理可以访问自己的参数和变量, 他也能**自由访问**把它嵌套在其中的**父函数**的参数与变量. 通过函数字面量创建的函数对象包含一个连接到外部上下文的连接, 叫做**闭包**.

```javascript
var add = function (a, b) {
  return a + b;
}
//or
function add (a, b) {
  return a + b;
}
```



## 调用

每个函数除了声明的变量外, 还会接收两个附加的参数`this`和`arguments`. 

调用运算符是跟在任何产生一个函数值的表达式之后的一对圆括号. 这就解释了立即执行函数的写法. 圆括号内包含参数. 参数个数不会导致运行时错误, 多余参数会被忽略, 参数缺失会被替换为`undefined`.

### 方法调用模式

当一个函数被保存为一个对象的属性时, 称之为`方法`. 

这种模式下`this`参数被绑定到该对象.

### 函数调用模式

当一个函数不是一个对象的属性时, 他就是被当作函数来调用的.

这种模式下`this`被绑定到全局对象. 作者认为这是一个语言设计上的错误, 应该将其绑定到外部函数的`this`变量. 我们可以这样做:

```javascript
var foo = function () {
  var that = this;
  var helper = function () {
    //do something with var that
  }
}
```

### 构造函数调用模式

使用`new`来调用一个函数时, 就会创建一个连接到该函数的`prototype`成员的新对象, 同时, `this`被绑定到这个新对象上.

```javascript
var Car = function () {
  this.wheels = 4;
}
var ford = new Car();
ford.__proto__ === Car.prototype //true
```

### `Apply`调用模式

`apply`方法让我们构建一个参数数组传递给调用函数. 同时, 也可以指定`this`的值.

该方法有两个参数:

- 第一个: 绑定给`this`的值
- 第二个: 参数数组

## 参数

前面提到过`arguments`参数, 函数可以通过访问此参数以访问所有参数, 包括多余参数. 然而, 因为语言的设计错误, 该参数只是一个"类似数组"的对象, 它拥有一个`length`属性, 但没有任何数组的方法.

## 返回

使用`return`关键字返回, 如果没有指定, 则返回`undefined`. 构造函数调用时, 如果返回值不是一个对象, 则返回`this`.

## 异常

抛出:

```javascript
var add = function (a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw {
      name: 'TypeError',
      message: 'add needs numbers'
    };
  }
  return a + b;
}
```

捕获:

```javascript
var try_it = function() {
  try {
    add("seven");
  } catch (e) {
    document.writeln(e.name + "; " + e.message);
  }
}
```

一个`try`语句只会有一个捕获异常的代码块, 如果有多种异常的情况, 只有通过`name`属性判断.

## 闭包

通过函数字面量创建的函数对象包含一个连接到外部上下文的连接, 叫做**闭包**.

因为`JavaScript`是一个函数式语言, 所以支持返回一个函数. 这样将会导致内部函数比它的外部函数拥有更长的生命周期. 这一特性也让创建私有变量成为可能.

```javascript
var myObj = (function () {
  var value = 1;
  return {
    setValue: function (inc) {
      value = typeof inc === 'number' ? inc : value;
    },
    getValue: function () {
      return value;
    }
  };
}());
```

再来看一个糟糕的例子及其改进:

```javascript
var add_the_handlers = function (nodes) {
  var i;
  for (i = 0; i < nodes.length; i++) {
    nodes[i].onclick = function (e) {
      alert(i);
    };
  }
};
```

这个函数的目的是点击一个节点时, 弹出对话框显示节点的序号. 而这个函数的效果却是每次显示节点的数目.

原因在于, 创建`onclick`函数时, 函数引用的变量`i`属于`add_the_handlers`方法, 而`i`一直在改变, 直到变为`nodes.length`. 所以, 当所有的`onclick`方法创建完成后, 引用的`i`实际上是一个变量, 值为`nodes.length`.

```javascript
var add_the_handlers = function (nodes) {
  var helper = function (i) {
    return function (e) {
      alert(i);
    };
  };
  var i;
  for (i = 0; i < nodes.length; i++) {
    nodes[i].onclick = helper(i);
  }
}
```

这个改进的方法就能达到目的, 原因在于, 返回给`onclick`的函数是`helper`的内部函数, 其引用的`i`是`helper`函数的`i`, 覆盖了外部的`add_the_handlers`的`i`. 所以, 当循环进行时, `helper`中的`i`因为是形参, 而不会收到`add_the_handlers`中的`i`的影响.

## 级联

其实这是一个技巧.

多数的`setter`方法往往不需要返回任何内容, 这时在`JavaScript`中, 函数将会返回`undefined`. 如果我们需要对一个对象设置很多值, 不得不写成:

```javascript
obj.setName(name);
obj.setAge(age);
obj.setSex(sex);
...
```

如果我们让这样的函数返回`this`, 就可以启动`级联`, 情况就大不一样.

```javascript
obj.setName(name).setAge(age).setSex(sex)...
```

