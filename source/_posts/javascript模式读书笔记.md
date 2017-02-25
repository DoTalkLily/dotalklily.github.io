---
layout: post
title:  Javascript设计模式 读书笔记
date:   2014-06-12 08:43:59
author: Lily
categories: frontend
tags:
- javascript设计模式
- 读书笔记
---

最近读了《Javascript设计模式》这本书，有个习惯是会把觉得有用的内容记下来，之后只看这部分内容 ：）

使用var创建的全局变量不能删除，不适用var创建的隐含全局变量可以删除。
遍历数组时注意缓存长度：
for(var i = 0,max = arr.length;i<max;i++){} =>var i = arr.length;while(i--){} 更有效率，因为和0比较更快。

for...in 遍历对象，如果想过滤原型属性，如clone，toString等 可以用hasOwnProerty，如：
for(var i in obj){
  if(obj.hasOwnProperty(i){
       ///
   }
}
或写成：
var i, hasOwn = Object.prototype.hasOwnProperty;
for(i in obj){
    if(hasOwn.call(obj,i){ }
}

new Function()中的代码将在局部函数空间中云心个，因此代码中任何采用var定义的变量不会自动成为全局变量。另一个避免自动成为全局变量的方法是将eval()调用封装到一个及时函数中。
var str = "var j = 1;console.log(j)";

1. eval(str);
2. new Function(str);
3. (function(){
      eval(str);
    }());

=>三个方法之后再执行console.log(j);只有第一个eval会定义成全局变量，另两个都是undefined；
另一个区别是 eval可以访问和修改外部作用域的变量，Function不行:
(function(){
    var a = 1;
    eval("a = 3;");    new Function("console.log(a);");=> undefined
    console.log(a); // a = 3
}());


parseInt 要注意！ES3中 0开头的字符串会被当成八进制，ES5正常，所以第二个表示进制的参数尽量指定。否则
parseInt("09") =>0
简单的做法是+"08"  或者 Number("08") 这种做法会比parseInt快，但是parseInt可以用在输入比较复杂的情况 如 parseInt("08 hahahah");

判断一个对象是否是数组 ES5引入Array.isArray();通用的写法是：
if(typeof Array.isArray === "undefined"){
    Array.isArray = function(arg){
        return Object.prototype.toString.call(arg) === "[object Array]";
    }
}

可以使用Number String Boolean 创建包装对象。但是一般使用字面量定义就够了 ，var str = 'str'; 除非是想扩充对象，比如 var greet = "hello"; greet.smile = true;这样写smile属性是不会被保留的，不报错，返回undefined，的那是用new String("hello") 初始化的变量就可以加属性并且被保留。但是不影响其他变量使用new String初始化。
没使用new，包装构造函数将传递给他们的参数转换成一个基本类型。
typeof Number(1) => "number"
typeof String(1) => "string"
typeof Boolean(1) => "boolean"
反之 typeof new xxx 都返回 “object”


js内置的错误构造函数 Error()、SyntaxError()、TypeError()以及其他都有name 和message两个属性，可以自定义错误处理方式。

```js
try{
     throw{
         name:"myerror",
         message:"opps",
         extra:"hi",
         errorHandle:handle//自定义错误处理函数
     };
}catch(e){
     alert(e.message);
     e.errorHandle();
}
```

静态方法：
var Person = function(){}
Person.isProgramer = function(){ }//静态方法 这里的this === Person this指向构造函数
Person.prototype.isKind = function(){}//实例方法 这里的this !== Person 是一个对象
typeof Person.isKind ; =>undefined
typeof new Person().isProgramer; =>undefined
静态方法不需要对象直接用 类名调用，实例方法需要一个对象才能运行。以静态方式调用实例方法无法运行，用对象.静态方法 方式调用静态方法也不能运行。

私有静态成员：
1. 同一个构造函数创建的所有对象共享该成员
2. 构造函数外部不可访问该成员
可以通过闭包实现，如：

```js
var Person = (function(){
     var counter = 0,
         RealPerson;
    RealPerson = function(){
        counter += 1;//私有静态成员变量
    };
    RealPerson.prototype.getId = function(){
        return counter;
    }
    return RealPerson;
}());
var p1 = new Person();
p1.getId(); =>1
var p2 = new Person();
p2.getId(); =>2
```

继承模式：

```js
var inherit = (function(){
    var F = function(){};
    return function(C,P){
        F.prototype = P.prototype;
        C.prototype = new F();
        C.uper = P.prototype;
        C.prototype.constructor = C;
    }
})());
```


通过复制实现继承
浅复制：

```js
function extend(parent,child){
    var i ;
    child = child || {};
    for( i in parent){
         if(parent.hasOwnProperty(i)){
             child[i] = parent[i];
         }
    }
return child;
}
```

深复制:

```js
function extendDeep(parent,child){
     var i,
         toString = Object.prototype.toString(),
         arrStr = "[object Array]";
     child = child || {};

     for(i in parent){
         if(typeof patrent[i] === "object"){
             child[i] = (toString.call(parent[i]) === arrStr)?[]:{};
             extendDeep(parent[i],child[i]);
         }else{
             child[i] = parent[i];
         }
     }
     return child;
}
```

混入，从多个对象中复制成员并组合成新的对象，相当于组合。

```js
function mix(){
     var arg,prop,child = {};
     for(arg = 0;arg <arguments.length;arg+=1){
         for(prop in arguments[arg]){
             if(arguments[arg].hasOwnProperty(prop)){
                child[prop] = arguments[arg][prop];
             }
         }
     }
     return child;
}
```

这里重复属性会被覆盖，可以做判断

```js
var cake = mix(
     {eggs:2},
     {milk:30}
);
```

借用方法，有时候只想用某个对象的一些方法，而不是继承全部属性和方法，这时候可以用call apply
yourObject.doStuff.call(myObj, p1,p2);
yourObject.doStuff.apply(myObj, [p1,p2]);
如：借用数组方法：

```js
function f(){
    var args = [].slice.call(arguments,1,2);//这里可以是Array.prototype.slice.call 可以减少空数组创建
    return args;
}
```

call  apply 和 bind区别：
When you want that function to later be called with a certain context, useful in events.

Call/apply call the function immediately, whereas bind returns a function that when later executed will have the correct context set for calling the original function. This way you can maintain context in async callbacks, and events.

I do this a lot:

```js
function MyObject(element) {
    this.elm = element;

    element.addEventListener('click', this.onClick.bind(this), false);};

MyObject.prototype.onClick = function(e) {
     var t=this;  //do something with [t]...
    //without bind the context of this function wouldn't be a MyObject
    //instance as you would normally expect.
};
```
I use it extensively in node.js for async callbacks that I want to pass a member method for, but still want the context to be the instance that started the async action.

A simple, naive implementation of bind would be like:

```js
Function.prototype.bind = function(ctx) {
    var fn = this;
    return function() {
        fn.apply(ctx, arguments);
    };};
```