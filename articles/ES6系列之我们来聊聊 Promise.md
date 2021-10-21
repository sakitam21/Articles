# ES6 系列之我们来聊聊 Promise

## 前言
Promise的基本使用可以看阮一峰老师的《ECMAScript6入门》
我们来聊点其他的。

## 回调
说起Promise，我们一般都会从回调或者回调地狱说起，那么使用回调到底会导致哪些不好的地方呢？
### 1.回调嵌套
使用回调，我们很有可能会将业务代码写成如下这种形式：
```javascript
  doA(function(){
    doB()
    doC(function(){
      doD()
    })
    doE()
  })
  doF()
```

当然这是一种简化的形式，经过一番简单的思考，我们可以判断出执行的顺序为：
```javascript
  doA()
  doF()
  doB()
  doC()
  doE()
  doD()
```

然而在实际的项目中，代码会更加杂乱，为了排查问题，我们需要绕过很多碍眼的内容，不断的在函数间跳转，使得排查问题的难度也在成倍增加。

当然之所以导致这个问题，其实是因为这种嵌套的书写方式跟人线性的思考方式相维和，以至于我们要多花一些精力去思考真正的执行顺序，嵌套和缩进只是这个思考过程中转移注意力的细枝末节而已。

当然了，与人线性的思考方式相违和，还不是最糟糕的，实际上，我们还会在代码中加入各种各样的逻辑判断，就比如在上面这个例子中，doD()必须在doC()完成后才能完成，万一doC()执行失败了呢？我们是要重试doC()吗？还是直接转到其他错误处理函数中？当我们将这些判断都加入到这个流程中，很快代码就会变得非常复杂，以至于无法维护和更新。

### 2.控制反转
正常书写代码的时候，我们理所当然可以控制自己的代码，然而当我们使用回调的时候，这个回调函数是否能接着执行，其实取决于使用回调的那个API，就比如：
```javascript
  //回调函数是否被执行取决于buy模块
  import { buy } from './buy.js'
  buy(itemData, function(res){
    console.log(res)
  })
```

**对于我们经常会使用的fetch这种API，一般是没有什么问题的，但是如果我们使用的是第三方的API呢？**

当你调用了第三方的API，对方是否会因为某个错误导致你传入的回调函数执行了多次呢？

为了避免出现这样的问题，你可以在自己的回调函数中加入判断，可是万一又因为某个错误这个回调函数没有执行呢？万一这个回调函数有时同步执行有时异步执行呢？

我们总结一下这些情况：

1.回调函数执行多次

2.回调函数没有执行

3.回调函数有时同步执行有时异步执行

对于这些情况，你可能都要在回调函数中做些处理，并且每次执行回调函数的时候都要做些处理，这就带来了很多重复的代码。

## 回调地狱
我们先看一个简单的回调地狱的示例。

现在要找出一个目录中最大的文件，处理步骤应该是：

1.用fs.readdir获取目录中的文件列表

2.循环遍历文件，使用fs.stat获取文件信息

3.比较找出最大文件

4.以最大文件的文件名为参数调用回调

代码为：
```javascript
  var fs = require('fs')
  var path = require('path')
  function findLargest(dir, cb) {
    //读取目录下的所有文件
    fs.readdir(dir, function(er, files){
      if(er) return cb(er)
      var counter = files.length
      var errored = false
      var stats = []
      files.forEach(function(file,index){
        //读取文件信息
        fs.stat(path.join(dir,file),function(er,stat){
          if(errored) return;
          if(er){
            errored = true;
            return cb(er);
          }
          stats[index] = stat
          //事先算好有多少个文件，读完1个文件信息，计数减1，当为0时，说明读取完毕，此时执行最终的比较操作
          if(--counter==0){
            var largest = stats
              .filter(function(stat) {return stat.isFile()})
              .reduce(function(prev,next){
                if(prev.size>next.size) return prev
                return next
              })
            cb(null,files[stats.indexOf(largest)])
          }
        })
      })
    })
  }
```

使用方式为：
```javascript
  //查找当前目录最大的文件
  findLargest('./',function(er,filename){
    if(err) return console.error(er)
    console.log('largest file was:',filename)
  })
```

你可以将以上代码复制到一个比如index.js文件，然后执行node index.js就可以打印出最大的文件的名称。

看完这个例子，我们再来聊聊回调地狱的其他问题：

### 1.难以复用
回调的顺序确定下来以后，想对其中的某些环节进行复用也很困难，牵一发而动全身。

举个例子，如果你想对fs.stat读取文件信息这段代码复用，因为回调中引用了外层的变量，提取出来后还需对外层的代码进行修改。

### 2.堆栈信息被断开
我们知道，JavaScript引擎维护了一个执行上下文栈，当函数执行的时候，会创建该函数的执行上下文压入栈中，当函数执行完毕后，会将该执行上下文出栈。

如果A函数中调用了B函数，JavaScript会先将A函数的执行上下文压入栈中，再将B函数的执行上下文压入栈中，当B函数执行完毕，将B函数执行上下文出栈，当A函数执行完毕后，将A函数执行上下文出栈。

这样的好处在于，我们如果中断代码执行，可以检索完整的堆栈信息，从中获取任何我们想获取的信息。

可是异步回调函数并非如此，比如执行fs.readdir的时候，其实是将回调函数加入任务队列中，代码继续执行，直至主线程完成后，才会从任务队列中选择已经完成的任务，并将其加入栈中，此时栈中只有这一个执行上下文，如果回调报错，也无法获取调用该异步操作时的栈中的信息，不容易判定哪里出现了错误。

此外，因为是异步的缘故，使用try catch语句也无法直接捕获错误。
（不过Promise并没有解决这个问题）

### 3.借助外层变量
当多个异步计算同时进行，比如这里遍历读取文件信息，由于无法预期完成顺序，必须借助外层作用域的变量，比如这里的count、errored、stats等，不仅写起来麻烦，而且如果你忽略了文件读取错误时的情况，不记录错误状态，就会接着读取其他文件，造成无谓的浪费。此外外层的变量，也可能被其它同一作用域的函数访问并且修改，容易造成误操作。

之所以单独讲讲回调地狱，其实是想说嵌套和缩进只是回调地狱的一个梗而已，它导致的问题远非嵌套导致的可读性降低而已。

## Promise
Promise使得以上绝大部分的问题都得到了解决。

### 1.嵌套问题
举个例子：
```javascript
  request(url, function(err,res,body){
    if(err) handleError(err)
    fs.writeFile('1.txt',body,function(err){
      request(url2,function(err,res,body){
        if(err) handleError(err)
      })
    })
  });
```

使用Promise后：
```javascript
  request(url)
  .then(function(result){
    return writeFileAsync('1.txt',result)
  })
  .then(function(result){
    return request(url2)
  })
  .catch(function(e){
    handleError(e)
  })
```

而对于读取最大文件的那个例子，我们使用Promise可以简化为：
```javascript
  var fs = require('fs');
  var path = require('path');

  var readDir = function(dir) {
    return new Promise(function(resolve, reject) {
        fs.readdir(dir, function(err, files) {
            if (err) reject(err);
            resolve(files)
        })
    })
  }

  var stat = function(path) {
    return new Promise(function(resolve, reject) {
        fs.stat(path, function(err, stat) {
            if (err) reject(err)
            resolve(stat)
        })
    })
  }

  function findLargest(dir) {
    return readDir(dir)
        .then(function(files) {
            let promises = files.map(file => stat(path.join(dir, file)))
            return Promise.all(promises).then(function(stats) {
                return { stats, files }
            })
        })
        .then(data => {
            let largest = data.stats
                .filter(function(stat) { return stat.isFile() })
                .reduce((prev, next) => {
                    if (prev.size > next.size) return prev
                    return next
                })

            return data.files[data.stats.indexOf(largest)]
        })
  }
```

### 2.控制反转再反转
前面我们讲到使用第三方回调API的时候，可能会遇到如下问题：

1.回调函数执行多次

2.回调函数没有执行

3.回调函数有时同步执行有时异步执行

对于第一个问题，Promise只能resolve一次，剩下的调用都会被忽略。

对于第二个问题，我们可以使用Promise.race函数来解决：
```javascript
  function timeoutPromise(delay){
    return new Promise(function(resolve,reject){
      setTimeout(function(){
        reject("Timeout!")
      },delay)
    })
  }
  Promise.race([
    foo(),timeoutPromise(3000)
  ]).then(function (){

  },function (err){

  })
```

对于第三个问题，为什么有的时候会同步执行有的时候会异步执行呢？

我们来看个例子：
```javascript
  var cache = {...};
  function getValue(url) {
    if(cache.has(url)) {
      // 如果存在cache，这里为同步调用
      return Promise.resolve(cache.get(url));
    }
    return fetch(url).then(file => cache.set(url, file)); // 这里为异步调用
  }
  console.log('1');
  getValue.then(() => console.log('2'));
  console.log('3');
```
在这个例子中，有cache的情况下，打印结果为1 2 3，在没有cache的时候，打印结果为1 3 2。

然而如果将这种同步和异步混用的代码作为内部实现，只暴露接口给外部调用，调用方由于无法判断是到底是异步还是同步状态，影响程序的可维护性和可测试性。

简单来说就是同步和异步共存的情况无法保证程序逻辑的一致性。