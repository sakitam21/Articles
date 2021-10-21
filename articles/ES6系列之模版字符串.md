# ES6 系列之模版字符串

## 基础用法
```javascript
  let message = `Hello World`;
  console.log(message);
```

如果你碰巧要在字符串中使用反撇号，你可以使用反斜杠转义：
```javascript
  let message = `Hello \` World`;
  console.log(message);
```

值得一提的是，在模板字符串中，空格、缩进、换行都会被保留：
```javascript
  let message = `
    <ul>
      <li>1</li>
      <li>2</li>
    </ul>
  `;
  console.log(message)
```

## 嵌入变量
模版字符串支持嵌入变量，只需要将变量名写在${}之中，其实不止变量，任意的JavaScript表达式都是可以的：
```javascript
  let x=1,y=2
  let message = `<ul><li>${x}</li><li>${y}</li></ul>`;
  console.log(message); // <ul><li>1</li><li>2</li></ul>
```

值得一提的是，模版字符串支持嵌套：
```javascript
  let arr = [{value: 1}, {value: 2}];
  let message = `
	  <ul>
		  ${arr.map((item) => {
			  return `
				  <li>${item.value}</li>
			  `
		  }).join('')}
	  </ul>
  `;
  console.log(message);
```

## 标签模板
模板标签是一个非常重要的能力，模板字符串可以紧跟在一个函数名后面，该函数将被调用来处理这个模板字符串，举个例子：
```javascript
  let x = 'Hi',y='Kevin';
  var res = message`${x}, I am ${y}`;
  console.log(res)
```