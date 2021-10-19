JavaScript专题之函数柯里化
==========================

定义
------------
维基百科中对柯里化（Currying）的定义为：在数学和计算机科学中，柯里化是一种将使用多个参数的一个函数转换成一系列使用一个参数的函数的技术。

举个例子：
```javascript
  function add(a, b) {
    return a + b
  }

  //执行add函数，一次传入两个桉树即可
  add(1,2) //3

  //假设有一个curry函数可以做到柯里化
  var addCurry = curry(add)
  addCurry(1)(2) //3
```

用途
-----------------
我们会讲如何写出这个curry函数，并且会将这个curry函数写的很强大，但是在编写之前，我们需要知道柯里化到底有什么用？

举个例子：
```javascript
  //示意而已
  function ajax(type,url,data){
    var xhr = new XMLHttpRequest()
    xhr.open(type,url,true)
    xhr.send(data)
  }

  //虽然ajax这个函数非常通用，但在重复调用的时候参数冗余
  ajax('POST','www.test.com',"name=kevin")
  ajax('POST','www.test2.com',"name=kevin")
  ajax('POST','www.test3.com',"name=kevin")

  //利用curry
  var ajaxCurry = curry(ajax)

  //以POST类型请求数据
  var post = ajaxCurry('POST')
  post('www.test.com',"name=kevin")

  //以POST类型请求来自于www.test.com的数据
  var postFromTest = post('www.test.com')
  postFromTest("name=kevin")
```

想象jQuery虽然有$.ajax这样通用的方法，但是也有$.get和$.post的语法糖。（当然jQuery底层是否这样做的，我就没有研究了）。

**curry的这种用途可以理解为：参数复用。本质上是降低通用性，提高适用性。**

可是即便如此，是不是依然觉得没什么用呢？

如果我们仅仅是把参数一个一个传进去，意义可能不大，但是如果我们把柯里化后的函数传给其他函数比如map呢？

举个例子：

比如我们有这样一段数据：
```javascript
  var person = [{name:'kevin'}, {name:'daisy'}]
```

如果我们要获取所有的name值，我们可以这样做：
```javascript
  var name = person.map(function(item) {
    return item.name
  })
```

不过如果我们有curry函数：
```javascript
  var prop = curry(function(key, obj) {
    return obj[key]
  })
  var name = person.map(prop('name'))
```

我们为了获取name属性还要再编写一个prop函数，是不是又麻烦了些？

但是要注意，prop函数编写一次后，以后可以多次使用，实际上代码从原来的三行精简成了一行，而且你看代码是不是更加易懂了？

`person.map(prop('name'))`就好像直白的告诉你：person对象遍历（map）获取（prop）name属性。

是不是感觉有点意思了呢？

第一版
-----------------
未来我们会接触到更多有关柯里化的应用，不过那是未来的事情了，现在我们该编写这个curry函数了。

一个经常会看到的curry函数的实现为：
```javascript
  //第一版
  var curry = function (fn) {
    var args = [].slice.call(arguments, 1)
    return function() {
      var newArgs = args.concat([].slice.call(arguments))
      return fn.apply(this, newArgs)
    }
  }
```
