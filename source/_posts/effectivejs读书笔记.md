---
layout: post
title:  Effective Javascript读书笔记
date:   2015-01-12 08:43:59
author: Lily
categories: frontend
tags:
- effective javascript
- 读书笔记
---

最近读了《Effective Javascript》这本书，有个习惯是会把觉得有用的内容记下来，之后只看这部分内容 ：）

1. 了解你使用的javascript版本
   为了保证程序在所有浏览器都运行正常，但是程序员没有办法指定js执行版本，es5引入‘严格模式’限制js中易出错或者问题较多的特性。可以在函数体的开始处和整个脚本的开始处加入“use strict”;如果一个页面引入多个js文件，有的是严格模式，有的不是，那么引入顺序会导致是否按照严格模式运行，严格模式先加载，那么都会按照严格模式运行，反之都按照非严格模式。避免这种问题的方法是从脚本级别或者函数级别封装到立即调用的函数表达式里以做到互不干扰。如：

    ```js
    (function(){
        "use strict";
         function a(){
         }
    })();
    ```

2. 理解javascript浮点数
   js黄宗所有数字都是双精度浮点数，位操作时比较特殊，js会将操作数转换成32位有符号整数后进行。浮点数运算注意精度问题，可能会有误差。尽量转换成整数再运算。

3. 当心隐式类型转换
    "- * /"会把左右两端的操作数转换成数字，+ 比较特殊，如果至少有一个字符串，结果就是字符串（内容和字符串顺序相关），if || && 判断传入 false 0 -0 undefined NaN null "" 会返回false，其他都为true，因此判断是否undefined用 if(typeof xx ==='undefined') 或 if(xx === undefined)
如果只重写了toString，对象转换时会无视valueOf的存在来进行转换。但是，如果只重写了valueOf方法，在要转换为字符串的时候会优先考虑valueOf方法。其他情况一般是有类型转换成字符串的时候会调用toString，进行运算符操作调用valueOf。

4. 原始类型优于封装对象
   javascript有5个原始类型，布尔，数字，字符串，null，undefined
   相等比较时，原始类型和封装对象表现不一样：
   var a = new String('aaa');
    var b = new String('aaa');
    a === b  =>false

    var a = 'aaa';
    var b = 'aaa';
    a === b =>true
    获取和设置原始类型属性会隐式地创建封装对象。不过设置一个自定义属性的值是不会被保留下来的。
    "hello".someProperty=17;
    "hello".someProperty; =>undefined

5. 避免对混合类型使用 == 运算符
    建议使用严格的等号运算符比较 === 以保证在比较过程中没有任何转换，否则执行一些转换规则
    null == undefined =>true  null === undefined =>false

    == 运算符的强制转换规则

    当比较不同类型的值时，用自定义的转换程序使程序的行为更清晰。

6. 了解分号插入的局限
   * 分号在}标记之前、一个或多个换行之后和程序输入的结果被插入。
     分号在随后的输入不能被解析时插入。如：
     a = b
     (f());    => a = b(f());
     如果某条语句的下一行的初始标记不能被解析为一个语句的延续，那么不能省略该条语句的分号。 如：
     a = b
     f(); 会报错

   * 在以 (  [  + - / 字符开头的语句不能省略分号，因为不会被自动插入分号。如：
      a = b
      ["r","g","b"].forEach(function(key){ });
      会被解析成 a= b["r","g","b"].forEach(function(key){ });
      / 可能在一些正则表达式中会出现在语句的开头
      a = b
      /Error/i.test(str) 被解析成  a = b/Error/i.test(str)  /变成除法运算符。

      a = b                 var x
      var x    不等于    a = b
      (f())                   (f())
      前者是三条语句，后者后两条合并。想省略分号时，可以在该语句后面加一个变量声明。为了保证安全在(f())前加一个；以保证三条语句顺序乱了也可以被解析。

   * 注意在脚本连接时可以在脚本之间显示地插入分号。
     ;(function(){})()

   * 在 return throw break continue ++ --的参数之前绝不要加入换行（break continue是带显式标签的）。
     a
     ++   => a; ++b;
     b

   * for条件内 和 空循环体语句不要缺少分号！ wihle(true);  否则报错。


闭包存储的是外部变量的引用，而不是副本，并能读写这些变量。

变量声明的提升，函数声明也会提升。js没有块级作用域只有函数级作用域，变量声明会提升到包含这个变量的函数顶部，但初始化仍停留在原地，例外！！！try catch语句将捕获的异常绑定到一个变量，该变量的作用域只是catch语句块。

```js
function test(){
   var x ="test", result = [];
   result.push(x);
   try{
      throw "exception"
   }catch(x){
      x  ="catch";
   }
   result.push(x);
   return result;   =>["test","test"]
}
```

arguments 严格模式下是实参的副本，非严格模式下是引用。[].slice.call(arguments) 将arguments复制到一个真正的数组中再修改比较安全。 arguments没有数组的某些函数，可以通过[].xxx.call(arguments)方式来对其进行操作。


```js
var buffer={
     entries:[],
     add:function(s){
          this.entries.push(s);
     },
     concat：function(){
          return this.entries.join("");
     }
}
var source = ["333","egerge"];
source.forEach(buffer.add); // 这里报entries undefined
```

因为this此时绑定的是全局对象。forEach参数第二个参数作为接受者 source.forEach(buffer.add,buffer)
或者用匿名函数
source.forEach(function(s){
    buffer.add(s);
});
或者用bind
source.forEach(buffer.add.bind(buffer));

将字符串传递给evel函数时不要在字符串中包含局部变量的引用。

函数toString 方法不会暴露存储在闭包中的局部变量的值。

```js
(function(x){
    return function(y){
          return x+y;
    }
})(40).toString();//functino(y){return x+y}
```

非严格模式下调用函数，函数中的this是全局对象，严格模式下是undefined。


```js
function C(){};
C.prototype = null;
var o = new C();
Object.getPrototypeOf(o) === null;//false;
Object.getPrototypeOf(o) === Object.prototype;//true

var x  =Object.create(null);
Object.getPrototypeOf(o) === null;//true;
```

for...in 循环枚举对象属性时与顺序无关。使用数组而不是字典存储有序集合。

如果要在一个对象上（如Object.prototype)添加属性并不会出现在for...in循环中，可以用es5的Obejct.defineProperty Object.defineProperties定义属性并将它的enumerable设为false（value,writable,enumerable,configurable)


```js
Object.defineProperty(Object.prototype, "newKey",{
    value:xxx,
    writable:true,
    enumerable:false,
    configurable:true
};
```

如果被枚举的对象在枚举期间添加了新的属性，那么在枚举期间并不能保证for...in 能访问到新添加的成员。因为for...in 枚举对象是无序的，可以考虑用while 和 for。


```js
var scores = [12,2,3,4,5,5,6,2,3];
var total  = 0;
for(var score in scores){
    total+=score;
}
var mean = total / scores.length;
```

这里不会得到正确的结果，因为for...in遍历 score此时是索引值，而且是字符串，这里total执行是的是字符串拼接。total得到0123456xxx 所以建议用传统的for循环而且条件部分建议写成
for(var i = 0,n=scores.length;i<n;i++) 以避免每次循环都重复计算一次score.length


类数组对象：
有一个范围在0 到 2^32-1的整型length属性。
length属性大于该对象的最大索引。索引是范围在0到2^32-2的整数
这就是一个对象需要实现与Array.prototype中任意方法兼容的所有行为。
var arrayLike = {0:"a",1:"b",length:2};
var result = Array.prototype.map.call(arrayLike,function(s){
    return s.toUpperCase();
});// ["A","B"]

字符串也可表现为不可变数组，因为它们是可索引的，并且长度可以通过length属性获取。
var result  =Array.prototype.map.call("abc",function(s){
    return s.toUpperCase();
});//["A","B","C"]
数组有两规则类数组比较难以实现：
1. 将length设置为较小的数会截断数组。
2. 插入一个索引大于length的元素会将length设置为索引+1
但是用Array.prototype中的方法，这两条规则都不是必须的，因为增加或删除索引属性的时候他们回强制更新length属性。
只有一个concat方法不是通用的。如果参数不是真实的数组，会将参数作为一个元素连接起来。
function namesColumn(){
    return ["names"].concat(arguments);
}
namesColumn("alice","lily");//["names",{0:"alice",1:"lily"}]

解决方法：
function namesColumn(){
    return ["names"].concat([].slice.call(arguments)) ;
}

使用选项对象使得API更具可读性，更容易记忆。
所有通过选项对象提供的参数应当被视为可选的。
使用extend函数抽象出的从选项对象中提取的逻辑。
var alert = new Alert(app,150,140,true,false......);//很多参数
可以提取必填的参数，将可选的参数放到对象中传递

```js
funtion Alert(parent,message,opts){
     opts = opts || {};
     this.width = opts.width === undefined ? 320:opts.width;
     this.height = opts.height === undefined ? 240:opts.height;
     this.x = ....
     this.y = ....
     this.xxx= ...
     ....
}//每个参数都做这种判断也很麻烦
```

用extend函数

```js
function Alert(parent,message,opt){
     opts = extend({
         width:320;
         height:240
     });

     opts.extend({
       //各种参数的默认值
     },opts);

     extend(this,opts);
}
```


判断是否是数组的函数 ：
ES5中的Array.isArray
Object.prototype.toString.call(x) === "[object Array]";

