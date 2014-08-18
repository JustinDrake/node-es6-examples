##ECMAScript 6 in Node.JS



此文主要用一些简单的例子来介绍ECMAScript6(以下用ES6代替)的一些可以在Node中应用的特性。不需要转译器或[shim](http://www.cnblogs.com/ziyunfei/archive/2012/09/17/2688829.html)就可以运行这些例子。


ES6的基本方案已确定。但是在正式发布前一些具体的语法会有改动,实验性的ES6在Node不一定会遵循最近的[草案](http://people.mozilla.org/~jorendorff/es6-draft.html)


假定不稳定版本分支0.11.x比稳定分支0.10.x有更好的ES6支持。为了简单的切换Node版本， 建议使用TJ的[n](https://github.com/visionmedia/n)去做版本管理。



只加标志`--harmony`可以使用大部分的ES6实验特性。至于v0.11.13，为了使用block scoping等例子，也需要加`--use_strict`标志。


 > 译者注:
 **在使用版本v0.11.9, 使用let语法时，必须使用`node --harmony --use_strict test.js`, 否则会报警`SyntaxError: Illegal let declaration outside extended mode`**

欢迎Pull requests,Enjoy~


## <a name='toc'> 內容目錄</a>
 1. [塊作用域](#block-scoping)
 2. [遍歷器](#generators)
 3. [代理](#proxies)
 4. [Maps and Sets數據結構](#maps-and-sets)
 5. [Weak Maps](#weak-maps)
 6. [Symbols](#symbols)


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

    for(let i = 0; i < n; i += 1) { // 在loop初始使用块作用域
        let temp = previous;
        previous = current;
        current = temp + current;
    }

    return current;
}
```


第三个例子， 嵌套循环

```javascript
//ES5: 一个变量名不能重用 
var counter = 0;
for (var i = 0; i < 3; i+= 1) {
    for (var i = 0; i < 3; i += 1) {
        counter += 1;    
    }     
}

console.log(counter); // 输出 "3"
```


```javascript
// ES6: 一个变量名可以重用
var counter = 0;
for (let i = 0; i < 3; i += 1) {
    for (let i = 0; i < 3; i += 1) {
        counter += 1;    
    }    
}

console.log(counter); 输出 "9"
```


ES6的设计者也为`let`生了个妹妹。关键词`const`声明块作用域内的*静态*变量。

```
const a = 'You shal remain constant!';

a = 'I wanna be free'; // SyntaxError: Assignment to constant variable;
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

最后， 不能用`let`在同一个块作用域内对相同的变量声明多次，否则会抛出语法错误。在相同的函数域内使用`var`对一个变量多次声明不会抛出错误。 

```
{
    let a;
    let a; // 语法错误: Variable 'a' has already been declared
}
```



##Generators(遍历器)

有了Generators,函数可以被"分片"执行。在每片的结尾暂停，在下一片开始恢复。Generators的语法与函数类似但是`function`被替代为`function*`.`yield`语句是控制程序顺序的。


```
function* argumentsGenerator() {
    for (let i = 0; i < arguments.length; i += 1) {
        yield arguments[i];    
    }
}
```


(注意虽然ES5中`yield`不是保留字符, 新的`function*`语法保证在ES6中非ES5函数使用“yield”作为变量将会出错)


Generators可以返回迭代器。遍历器即带有`next`方法的对象，按顺序执行generators的主体。`next`方法， 被重复调用时， 其实是部分执行对应的产生器， 最终执行整个产生器主直到遇到`yield`关键字。


```
var argumentsIterator = argumentsGenerator('a', 'b', 'c');

// 输出 "a b c"
console.log(
    argumentsIterator.next().value,
    argumentsIterator.next().value,
    argumnetsIterator.next().value
);
```

只要对象的产生器主体还未`return`,迭代器的`next`方法返回一个带有`value`属性和`done`属性的对象。`value`属性就是被返回的值(或者说产生的值)。产生器主体未`return`前，`done`属性为`false`。产生器`return`时， `done`为`true`。如果`done`为`true`, `next`方法被调用，将会抛出错误。 


ES6有一些语法糖在遍历器上。

```
// 输出 "a","b","c"
for(let value of argumentsIterator) {
    console.log(value);    
}
```


有了Generators可以定义不能确定长度的队列....


```
function* fibonacci() {
    let a = 0; b = 1;
    while(true) {
        yield a;    
        [a, b] = [b, a + b];
    }
}
```


可以优雅的遍历其值。

```
// 遍历Fibonacci数字
for(let value of fibonacci(){
    console.log(value);    
})
```


Generators可以用来解决传统的嵌套式的控制流带来的烦恼，避免多重回调。 两个库， [task.js](https://github.com/mozilla/task.js)和[gen-run](https://github.com/creationix/gen-run)，可以帮助你用同步的风格来写异步Javascript。


```
// task.js example
spawn(function*(){
    var data = yield $.ajax(url);    
    $('#result').html(data);
    var status = $('#status').html('Download complete');
    yield status.fadeIn().promise();
    yield sleep(2000);
    status.fadeOut();
});
```

顺序控制流可以用`try`-`catch`语句， 可以预见在ES6通过回调去错误处理会慢慢减少。


使用`yield*`，可以让产生器`yield`一个遍历器。


```
let delegatedIterator = (function*(){
    yield 'Hello!';
    yield 'Bye';
}());

let delegatingIterator = (function* (){
    yield 'Greetings!';
    yield* delegatedIterator;
    yield 'OK, bye';
}());

// 输出 "Greetings!", "Hello!", "Bye!", "Ok, bye."
for(let value of delegatingIterator) {
    console.log(value);    
}
```


##Proxies


Proxy可以理解为一个园编程对象， 将原生的对象行为用函数调用来代替。包裹里的方法是与其相关的处理对象。

```
var random = Proxy.create({
    get: function () {
        return Math.random();
    }    
})
```


函数`Proxy.create`创建一个'代理',其处理对象传给第一个参数。这里我们用`get`函数包裹去改写读属性。每次`random`,会有一个新的随机值。

```
// 输出3个随机数

console.log(random.value, random.value, random.value);
```


相似的， 赋值属性可以再创建`set`函数包裹


```
var time = Proxy.create({
    get: function () {
        return Date.now();    
    },
    set: function () {
        throw 'Time travel error1';    
    }
})
```


继续阐述对象怎样表现天然属性，我们创建一个数组，其可以像Python一样可以通过负索引取值。


```
function pythonArray(array){
    var dummy = array;    
    return Proxy.create({
        set: function (receiver, index, value) {
            dummy[index] = value;    
        },
        get: function (receiver, index) {
            index = parseInt(index);
            return index < 0 ? dummy[dummy.length + index] : dummy[index];
        }
    })
}
```


现在索引`-1`引用数组最后一个元素， `-2`为倒数第二个数，等等


注意`set`有3个元素；`receiver`指代理， `index`为属性名， `value`为属性值。

```
// 输出 "gamma"
console.log(pythonArray(['alpha', 'beta', 'gamma'])[-1]);
```


代理也可以用来清理数据绑定。用Backbon.JS 模型, 例如数据绑定结束的标志是必须使用`model.get`和`model.set`方法。这种句法结构在proxies是不必要的。


下面用一个复杂的安全例子来结束proxies。


假设一个函数`f`想要共享一个对象`o`给另外一个函数`g`。

TODO(understand the example)


##Maps和sets数据结构

map可以把它看作一个对象，其与普通对象的不同之处在于， 它的键值可以为任意的对象。在ES5中，`toString`隐式调用属性键值在属性获得值之前， 考虑到`({}.toString())`结果为`[Object Object]`, 可以将属性键值给为对象键值


下面举例来说

```
const gods = [
    { name: 'Brendan Eich' },
    { name: 'Guido van Rossum' },
    { name: 'Raffaele Esposito' }
];


let miracles = new Map();

miracles.set(gods[0], 'Javascript');
miracles.set(gods[1], 'Python');
miracles.set(gods[2], 'Pizza Margherita');

// 输出 "Javascript"
console.log(miracles.get(gods[0]));
```


set是一种包含有限元素集合的数据结构。每个值只能出现一次。构造函数是`Set`, api很简单


```
// 输出 ['constructor', 'size', 'add', 'has', 'delete', 'clear']
console.log(Object.getOwnPropertyNames(Set.prototype));
```


为了举例验证， 我们调查了6个人生活中最快乐的事。

```
let surveyAnswers = ['sex', 'sleep', 'sex', 'sun', 'sex', 'cinema'];
let pleasures = new Set();
surveyAnswers.forEach(function(pleasure) {
    pleasures.add(pleasure);    
});
// 输出pleasures的数目 4, 排除了重复值
console.log(pleasures.size);
```


不幸的是，目前只支持array和set间的转换，set 遍历还未支持。

maps和sets本该是两种正常使用的数据结构， 但是一直是用对象来代替的。下面将讨论weak maps,这种数据结构是不能用ES5去实现的。


##Weak maps


Weak maps 相对于maps和sets已经是两个不同的概念了，因为其从根本来上来说已经不仅仅是语法糖了,Weak maps 像maps但是与垃圾回收紧密相连，它提供了一个工具可以让写出来的代码不会内存泄漏.


Node的Javascript虚拟机， V8定期释放不在作用域里的对象。 如果当前作用域没有对其的引用， 可以说这个对象不再在作用域内。如果这个对象在作用域内， 但是不再使用， 就认为是内存泄漏。这样的内存泄漏周期性的重复时， V8不断地给其分配内存， 最终会让程序崩溃。 


在weak map里设定一对键值，属性键值并没有引用。键名是对象的弱引用，意味着从v8垃圾回收的角度考虑， 他们是被忽略的。表现出来的特征, 键名是不可枚举的。weak maps也没有`size`属性，而其与maps很类似， 说明这些特性与v8垃圾回收忽略有关。


```
let weakMap = new WeakMap();

// 这句实际上是个空指令， 键值会立即被回收
weakMap.set({}, 'noise');
```


WeakMap的应用在于， 键所对应的对象被回收后， WeakMap自动移除对应的键值对。


```
let userMetadata = new WeakMap();

server.on('userConnect', function(user){
    userMetadata.set(user, { connectionTime: Date.now() });    
    server.on('userDisconnect', function() {
        使用user对象， 然后忽略它， `metadata`自动被抛弃    
    });
});
```


因为字面量直接赋值，所以字面量是不可以作为weak map的键名。

```
let weakMap = new WeakMap();

// TypeError: Invalid value used as weak map key
weakMap.set('example string literal', 0);
```
Javascript代码，没有方法确定对象是否被回收。要么选择相信虚拟机是对的， 或者, 虚拟机可以用debugger工具调试， [node-inspector](https://github.com/node-inspector/node-inspector) 。


##Symbols


Symbols 扩展了键名值的范围， 例如 对象属性标志符。在ES5中， 键名值明确对应了字符窜集合。

```
let a = {};
let debugSymbol = Symbol();
a[debugSymbol] = 'This property value is indentified by a symbol';
```

 > 译者注: 上例可以看出，Symbols是和String并列的， 是一种原始属性。下例验证


 ```
 a = Symbol();
 // 输出 symbol
 console.log(typeof(a));
 ```


Symbols构造函数决定了它是独一无二的， 这样也避免了命名冲突。唯一的意思可以理解为: 同时创建的两个symbol绝不会引用相同的对象属性。


在ES5中， 命名冲突只能靠不用相同的命名来解决。下面的代码来自Jquery源码也说明了这点

```
jQuery.extend({
  // 页面里每次的jQuery的复制都是独一无二的。
  // 非数字被移走为了匹配
  expando: "jQuery" + (core_version + Math.random()).replace(/\D/g, "");
});
```


唯一性得到了些有趣的结果。


```
let a = Map();
a.set(Symbol(), 'Noise');
// 输出 1， 虽然直到有一个元素， 但是这个元素不可能取得出来
console.log(a.size);
```


为了清楚的说明symbol, 给symbol赋一个不变的名字。

```
let testSymbol = Symbol('This is a test');
// 输出 "This is a test"
console.log(testSymbol.name);
```


TODO: 讨论私有symbols的应用


##Object observation （对象）


TODO



##Typed arrays and array buffers(数组)


TODO



##总结


直到v0.11.9, 很多ES6特性已经可以用了，包括 symbols ， weakmaps, proxies。 我们也期望即将到来的如 rest, spread操作符， 重组和类可以被加入Nodejs...





