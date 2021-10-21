# ES6系列之let和const

## 块级作用域的出现
通过var声明的变量存在变量提升的特性:
```javascript
  if(condition){
    var value = 1;
  }
  console.log(value)
```

初学者可能会觉得只有condition为true的时候，才会创建value，如果condition为false，结果应该是报错，然而因为变量提升的原因，代码相当于：
```javascript
  var value;
  if(condition){
    value = 1;
  }
  console.log(value)
```
**注意：变量的变量提升只提升声明；函数的变量提升声明和赋值都会被提升。**

如果condition为false，结果会是undefined。

除此之外，在for循环中：
```javascript
  for(var i =0;i<10;i++){
    // ...
  }
  console.log(i) //10
```

即使循环已经结束了，我们仍然可以访问i的值。

为了加强对变量声明周期的控制，ECMAScript6引入了块级作用域。

块级作用域存在于：

1.函数内部

2.块中（字符{和}之间的区域

## let和const
块级声明用于声明在指定块的作用之外无法访问的变量。

let和const都是块级声明的一种。

我们来回顾下let和const的特点：

### 1.不会被提升
```javascript
  if(false){
    let value = 1;
  }
  console.log(value); //Uncaught ReferenceError: value is not defined
```

### 2.重复声明报错
```javascript
  var value = 1;
  let value = 2; // Uncaught SyntaxError: Identifier 'value' has already been declared
```

### 3.不绑定全局作用域
当在全局作用域中使用var声明的时候，会创建一个新的全局变量作为全局对象的属性。
```javascript
  var value = 1;
  console.log(window.value);  // 1
```

然而let和const不会：
```javascript
  let value = 1;
  console.log(window.value);  // undefined
```

再来说下let和const的区别：

const用于声明常量，其值一旦被设定不能再修改，否则会报错。

**值得一提的是：const声明不允许修改绑定，但允许修改值。这意味着当用const声明对象时：**
```javascript
  const data = {
    value:1
  }
  // 没有问题
  data.value = 2;
  data.value = 3;
  // 报错
  data = {}; // Uncaught TypeError: Assignment to constant variable. 
```

## 临时死区
临时死区（Temporal Dead Zone），简写为TDZ。

let和const声明的变量不会被提升到作用域顶部，如果在声明之前访问这些变量，会导致报错：
```javascript
  console.log(typeof value); // Uncaught ReferenceError: value is not defined
  let value = 1;
```

这是因为JavaScript引擎在扫描代码发现变量声明时，要么将它们提升到作用域顶部（遇到var声明），要么将声明放在TDZ中（遇到let和const声明）。访问TDZ中的变量会触发运行错误。只有执行过变量声明语句后，变量才会从TDZ中移出，然后方可访问。

看似很好理解，不保证你不犯错：
```javascript
  var value = "global";
  // 例子1
  (function() {
    console.log(value);
    let value = 'local';
  }());
  // 例子2
  {
    console.log(value);
    const value = 'local';
  };
```
两个例子中，结果并不会打印 global ，而是报错`Uncaught ReferenceError: value is not defined` ，就是因为TDZ的缘故。

## 循环中的块级作用域

```javascript
  var funcs = []
  for(var i=0;i<3;i++){
    func[i]=function(){
      console.log(i)
    }
  }
  funcs[0](); //3
```

一个老生常谈的面试题，解决方案如下：
```javascript
  var funcs = []
  for(var i=0;i<3;i++){
    func[i]=(function(){
      console.log(i)
    }(i))
  }
  funcs[0](); //0
```

ES6的let为这个问题提供了新的解决方法：
```javascript
  var funcs = []
  for(let i=0;i<3;i++){
    func[i]=function(){
      console.log(i)
    }
  }
  funcs[0](); //0
```

问题在于，上面讲了let不提升，不能重复声明，不能绑定全局作用域等等特性，可是为什么在这里就能正确打印出i值呢？

如果是不重复声明，在循环第二次的时候，又用let声明了i，应该报错呀，就算因为某种原因，重复声明不报错，一遍一遍迭代，i的值最终还是应该是3呀，还有人说for循环的设置循环变量的那部分是一个单独的作用域，就比如：
```javascript
  for(let i=0;i<3;i++){
    let i = 'abc'
    console.log(i)
  }
  // abc
  // abc
  // abc
```

这个例子是对的，如果我们把let改成var呢？
```javascript
  for(var i=0;i<3;i++){
    var i = 'abc'
    console.log(i)
  }
  // abc
```

为什么结果就不一样了呢，如果有单独的作用域，结果应该是相同的呀······

如果要追究这个问题，就要抛弃掉之前所讲的这些特性！这是因为let声明在循环内部的行为是标准中专门定义的，不一定就与let的不提升特性有关，其实，在早期的let实现中就不包含这一行为。

而在for循环中使用let和var，底层会使用不同的处理方式。

那么当使用let的时候底层到底时怎么做的呢？

简单的来说，就是在`for(let i=0;i<3;i++)`中，即圆括号之内建立一个隐藏的作用域，这就可以解释为什么：
```javascript
  for (let i = 0; i < 3; i++) {
    let i = 'abc';
    console.log(i);
  }
  // abc
  // abc
  // abc
```

然后每次迭代循环时都创建一个新变量，并以之前迭代中同名变量的值将其初始化。这样对于下面这样一段代码
```javascript
  var funcs = [];
  for (let i = 0; i < 3; i++) {
    funcs[i] = function () {
        console.log(i);
    };
  }
  funcs[0](); // 0
```

就相当于：
```javascript
  // 伪代码
  (let i = 0) {
    funcs[0] = function() {
        console.log(i)
    };
  }

  (let i = 1) {
    funcs[1] = function() {
        console.log(i)
    };
  }

  (let i = 2) {
    funcs[2] = function() {
        console.log(i)
    };
  };
```

当执行函数的时候，根据词法作用域就可以找到正确的值，其实你也可以理解为let声明模仿了闭包的做法来简化循环过程。

## Babel

在Babel中是如何编译let和const的呢？我们来看看编译后的代码：
```javascript
  let value = 1;
  {
    let value = 2;
  }
  value = 3;
```

编译为：
```javascript
  var value = 1
  {
    var _value = 2;
  }
  value = 3;
```

本质是一样的，就是改变量名，使内外层的变量名称不一样。

## 最佳实践
在我们开发的时候，可能认为应该默认使用let而不是var，这种情况下，对于需要写保护的变量要使用const。然而另一种做法日益普及：默认使用const，只有当确实需要改变变量的值的时候才使用let。这是因为大部分的变量的值在初始化后不应再改变，而预料之外的变量的改变是很多bug的源头。
