##ECMAScript 6 in Node.JS


此文主要用一些简单的例子来介绍ECMAScript6(以下用ES6代替)的一些可以在Node中应用的特性。不需要转译器或[shim](http://www.cnblogs.com/ziyunfei/archive/2012/09/17/2688829.html)就可以运行这些例子。


ES6的基本方案已确定。但是在正式发布前一些具体的语法会有改动,实验性的ES6在Node不一定会遵循最近的[草案](http://people.mozilla.org/~jorendorff/es6-draft.html)


在Node中可以使用`node --v8-options | grep harmony`找到可以用的命令。不稳定版本分支0.11.x比稳定分支0.10.x有更好的ES6支持。为了简单，我们就用0.11.x分支， 但是大部分例子在0.10.x中也能工作。为了简单的切换Node版本， 建议使用TJ的[n](https://github.com/visionmedia/n)去做版本管理。



单独使用`--harmony`可以使用大部分的ES6实验特性。至于v0.11.3，为了使用block scoping 例子，也需要加`--use_strict`标志, 测试generator例子时，需要加`--harmony_generators`。


 > 译者注:
 **在使用版本v0.11.9, 使用let语法时，必须使用`node --harmony --use_strict test.js`, 否则会报警`SyntaxError: Illegal let declaration outside extended mode`**

欢迎Pull requests,Enjoy~


##Block scoping(块作用域)


先看看`let`。你可以把`let`理解为`var`的块作用域变量，功能和var一样，是声明变量，只是`let`声明的变量只在它所在的代码块里有效。

```
{
    let a = 'I am declared inside an anonymous block';
    console.log(a); // ReferenceError: a is not defined
}
```


ES6之前，Javascript 只有函数作用域。这其实是开发者hack的做法。下面将用两个例子来看看ES6带来的改进。


第一个例子是关于私有变量。

```
// ES5: 复杂的函数闭包 
var login = (function ES5() {
   var privateKey = Math.random();

   return function(password) {
        return password === privateKey; 
       };
}())
``` 
```
// ES6: 简单的块
{
   let privateKey = Math.random(); 
   var login = function(password) {
        return password === privateKey; 
       };
}
```

第二个例子

```
//ES5:  DEfensive declarations at the top to avoid hoisting surprise

function fibonacci(n) {
   var previous = 0; 
   var current = 1;
   var i;
   var temp;

   for(i = 0; i < n; i += 1){
        temp = previous;
        previous = current;
        current = temp + current;
    }
    return current;
}
```


```
// ES6: variables are concdaled within the apporpriate block scope
function fibonacci(n) {
    let previous = 0;    
    let current = 1;

    for(let i = 0; i < n; i += 1) {
        let temp = previous;
        previous = current;
        current = temp + current;
    }

    return current;
}
```


最后的那个例子`for`循环有个明显的块作用域。块作用域包含`i`的声明,同时循环遍历运行时块作用域被创建。


ES6的设计者也为`let`生了个妹妹。关键词`const`声明块作用域内的*静态*变量。

```
const a = 'You shal remain constant!';

// SyntaxError: Assignment to constant variable;
a = 'I wanna be free';
```


最后，ES6通过块作用域函数定义解决了一个疑难问题。 下面的代码在ES5中不能很好的定义。


```
function f() { console.log('I am outside'); }
(function () {
    if (false) {
        // 重新声明会怎么样?
        function f() { console.log('I am inside '); }
    }    
    f();
}());
```

`f`二次声明被销毁？`if`块没有执行导致其被忽略？其作用域在`if`块里？ 不同的浏览器有不同的表现。在ES6函数声明是块作用域，所以上面的代码输出`I am outside!`。

最后， 不能用`let`在同一个块作用域内对相同的变量声明多次，否则会抛出语法错误。在相同的函数域内使用`var`对一个变量多次声明不会抛出错误， 让很多开发者犯错。

```
var counter = 0;
for(var i = 0; i < 3; i += 1) {
    for(var i = 0; i < 3; i += 1) {
        counter += 1;    
    }
}
// 输出"3", 可能作者本意为9
console.log(counter);
```



##Generators(迭代器)


迭代器允许
