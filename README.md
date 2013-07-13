ECMAScript 6 in Node.JS
===

This text introduces and illustrates, with simple examples, ECMAScript 6 (ES6 for short) features natively available in Node. No transpiler or shim is required to run the examples. We hope the reader will find the subset of ES6 presented here interesting.

The underlying philosophy and broad direction of ES6 has mostly been agreed upon. However, implementation details have and will change until the final specification is published. Experimental ES6 in Node may not comply with the latest draft specification (which is available [here](http://people.mozilla.org/~jorendorff/es6-draft.html) in HTML format).

For a list of flags enabling experimental JavaScript in Node use the command `node --v8-options | grep harmony`. The unstable branch 0.11.x has greater ES6 support than the stable branch 0.10.x. For simplicity, we assume the 0.11.x branch here, but most examples will work in 0.10.x also. To easily switch between versions, we recommend the excellent [n](https://github.com/visionmedia/n) Node version manager.

The single `--harmony` flag enables most of the ES6 experimental features. However, as of v0.11.3, you will also need the `--use_strict` flag for the block scoping examples, along with the `--harmony_generators` flag for the generator examples.

Pull requests are welcome. Enjoy.

Block scoping
---
Let's start with `let`. You can think of `let` as a block-scoped variation of `var` for variable declaration.

```javascript
{ let a = 'I am declared inside an anonymous block'; }
console.log(a); // ReferenceError: a is not defined
```

Up until ES6, JavaScript only had function scoping. This is considered a design flaw which developers hack around. We illustrate improvements brought by ES6 with two examples.

The first example is about private variables.

```javascript
// ES5: A convoluted function closure
var login = (function ES5() {
  var privateKey = Math.random();

  return function (password) {
    return password === privateKey;
  };
}());
```
```javascript
// ES6: A simple block
{
  let privateKey = Math.random();

  var login = function (password) {
    return password === privateKey;
  };
}
```

The second example has to do with variable hoisting.

```javascript
// ES5: Defensive declarations at the top to avoid hoisting surprises
function fibonacci(n) {
  var previous = 0;
  var current = 1;
  var i;
  var temp;

  for(i = 0; i < n; i += 1) {
    temp = previous;
    previous = current;
    current = temp + current;
  }

  return current;
}
```
```javascript
// ES6: Variables are conceilled within the appropriate block scope
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

The `for` loop of the last example has an implicit block scope for the head. This block scope contains the declaration of `i` as well as the block scopes created at runtime by the loop iteration.

The designers of ES6 conceived of a baby sister for `let`. The keyword `const` declares block-scoped *constant* variables.

```javascript
const a = 'You shall remain constant!';

// SyntaxError: Assignment to constant variable
a = 'I wanna be free!';
```

Finally, ES6 is fixing a long standing issue with block scope function definitions. The following code is not well defined in the ES5 specification.

```javascript
function f() { console.log('I am outside!'); }
(function () {
  if(false) {
    // What should happen with this redeclaration?
    function f() { console.log('I am inside!'); }
  }

  f();
}());
```

Should the redeclaration of `f` be hoisted? Should it be ignored because the `if` block is not executed? Should it be scoped to the `if` block? Different browsers handle things differently. In ES6 function declarations are block-scoped, so the above snippet will print `I am outside!`.

Finally, ES6 throws a syntax error when multiple `let` declarations of the same variable occur in the same block. No analogous error is thrown for `var` redeclarations within the same function scope, which has led some developers astray.

```javascript
var counter = 0;
for(var i = 0; i < 3; i += 1) {
  for(var i = 0; i < 3; i += 1) {
    counter += 1;
  }
}

// Prints "3" although the author probably meant it to print "9"
console.log(counter);
```

Generators
---
Generators allow for function-like behaviour where execution is segmented into "pieces". Execution is paused at the end of each piece and can be resumed at the start of the next piece. The syntax for generators is similar to that of functions but the `function` keyword is replaced by `function*`. Flow control is dictated with `yield` statements.

```javascript
function* argumentsGenerator() {
  for (let i = 0; i < arguments.length; i += 1) {
    yield arguments[i];
  }
}
```

(Note that although the `yield` keyword is *not* a reserved keywork in ES5, the new `function*` syntax guarantees no ES5 function using "yield" as a variable name will break in ES6.)

Generators are useful because they return (i.e. create) iterators. In turn, an iterator, an object with a `next` method, actually executes the body of generators. The `next` method, when repeatedly called, partially executes the corresponding generator, gradually advancing through the body until a `yield` keyword is hit.

```javascript
var argumentsIterator = argumentsGenerator('a', 'b', 'c');

// Prints "a b c"
console.log(
    argumentsIterator.next().value,
    argumentsIterator.next().value,
    argumentsIterator.next().value
);
```

The `next` method of an iterator returns an object with a `value` property and a `done` property, as long as the body of the corresponding generator has not `return`ed. The `value` property refers the value `yield`ed or `return`ed. The `done` property is `false` up until the generator body `return`s, at which point it is `true`. If the `next` method is called after `done` is `true`, an error is thrown.

Alongside generators and iterators, ES6 has syntactic sugar for iteration.

```javascript
// Prints "a", "b", "c"
for(let value of argumentsIterator) {
  console.log(value);
}
```

Generators are ideal for defining sequences of undetermined lengths...

```javascript
function* fibonacci(limit) {
  let previous = 0;
  let current = 1;

  while(previous + current < limit) {
    let temp = previous;
    previous = current;
    yield current = temp + current;
  }
}
```

...which are elegantly enumerated.

```javascript
// Prints the Fibonacci numbers less than 1000
for(let value of fibonacci(1000)) {
  console.log(value);
}
```

Generators can be used to provide an alternative to the traditional nested callback flow control, thereby avoiding callback "pyramids" and "hell". Two libraries, [task.js](https://github.com/mozilla/task.js) and [gen-run](https://github.com/creationix/gen-run), aim to help write asynchronous JavaScript in a sequential style.

```javascript
// task.js example
spawn(function*() {
  var data = yield $.ajax(url);
  $('#result').html(data);
  var status = $('#status').html('Download complete.');
  yield status.fadeIn().promise();
  yield sleep(2000);
  status.fadeOut();
});
```

The sequential flow control also allows for meaningful `try`-`catch` statements, so the burden of explicitly passing errors through the callback chain, commonly found in Node libraries, can be alleviated in ES6.

To conclude, it is possible for a generator to `yield` to an iterator using a "delegated `yield`" with the syntax `yield*`.

```javascript
let delegatedIterator = (function* () {
  yield 'Hello!';
  yield 'Bye!';
}());

let delegatingIterator = (function* () {
  yield 'Greetings!';
  yield* delegatedIterator;
  yield 'Ok, bye.';
}());

// Prints "Greetings!", "Hello!", "Bye!", "Ok, bye."
for(let value of delegatingIterator) {
  console.log(value);
}
```

Proxies
---

A proxy is a meta-programming object for which some primitive object behaviours are replaced by function calls, called traps. The traps are methods in an associated handler object.

```javascript
var random = Proxy.create({
  get: function () {
    return Math.random();
  }
});
```

The function `Proxy.create` creates a proxy whose handler object is passed as the first argument. Here, we have a single `get` trap which overrides property reads. For each read of `random`, a new random value is computed at run time.

```javascript
// Prints three random numbers
console.log(random.value, random.value, random.value);
```

Similarly, property writes can be trapped by the `set` trap.

```javascript
var time = Proxy.create({
  get: function () {
    return Date.now();
  },
  set: function () {
    throw 'Time travel error!';
  }
});
```

The continue with the theme of dictating how objects should behave at the "native" level, we build arrays for which negative indexes behave as in Python.

```javascript
function pythonArray(array) {
  var dummy = array;

  return Proxy.create({
    set: function (receiver, index, value) {
      dummy[index] = value;
    },
    get: function (receiver, index) {
        index = parseInt(index);
        return index < 0 ? dummy[dummy.length + index] : dummy[index];
    }
  });
}
```

Now the index `-1` references the last array element, `-2` the penultimate, etc.

Notice `set` has three arguments; `receiver` refers to the proxy, `index` to the property name and `value` to the property value.

```javascript
// Prints "gamma"
console.log(pythonArray(['alpha', 'beta', 'gamma'])[-1]);
```

Proxies can also be used for clean data binding. With Backbone.JS models, for example, data binding is done at the expense of having to use the `model.get` and `model.set` methods. This syntactic indirection is unnecessary with proxies.

Let's conclude proxies with a somewhat sophisticated security example. Please put your abstraction hat on.

Suppose a function `f` wants to share an object `o` to another function `g` and later revoke access to `o`. Well, `f` can give `g` a proxy `p` to `o`. The traps of `p` grant or deny access based on a key `k` private to `f`. Actually, a proxy `q` on the handler object `h` of `p` can implement the access mechinism for all the traps of `p` simultaneously with a single `get` trap.

To recap, `g` interfaces through `p` to access `o`. The relevant trap of `p` in `h` is called triggering the `get` trap of `q` in charge of access control based on `k`.

TODO: Add examples which are not `get` or `set`.

Maps and sets
---

A map can be thought of as a object for which the keys can be arbitrary objects. In ES5, the method `toString` is implicitly called on property keys before a property access, which is less than helpfull for object keys given that `({}.toString())` is `[object Object]`.

Let's illustrate...

```javascript
const gods = [
  {name: 'Brendan Eich'},
  {name: 'Guido van Rossum'},
  {name: 'Raffaele Esposito'}
];

let miracles = Map();

miracles.set(gods[0], 'JavaScript');
miracles.set(gods[1], 'Python');
miracles.set(gods[2], 'Pizza Margherita');

// Prints "JavaScript"
console.log(miracles.get(gods[0]));
```

A set is a data structure containing a finite set of elements, each occuring exactly once. The constructor is `Set`, and the API is simple.

```javascript
// Prints [ 'constructor', 'size', 'add', 'has', 'delete', 'clear' ]
console.log(Object.getOwnPropertyNames(Set.prototype));
```

To illustrate, we have surveyed six people regarding their greatest pleasure in life...

```javascript
let surveyAnswers = ['sex', 'sleep', 'sex', 'sun', 'sex', 'cinema'];

let pleasures = Set();
surveyAnswers.forEach(function (pleasure) {
  pleasures.add(pleasure);
});

// Prints the number of pleasures in the survey, not counting duplicates
console.log(pleasures.size);
```

Unfortunately, support for array to set conversion, as well as set iteration is not yet supported in Node.JS.

Both maps and sets are naturally occuring data structures which can and have been implemented through object abuse. We now discuss weak maps, which are data structures which *cannot* be emulated with ES5.

Weak maps
---

Weak maps fit in a somewhat different category to that of maps and sets because weak maps fundamentally provide more than just syntax sugar. Weak maps are like maps but are intimately linked to garbage collection, providing a tool to help writing code that does not leak memory.

The JavaScript virtual machine, V8 in the case of Node, periodically frees memory allocated to objects no longer in scope. An object is not longer in scope if there is no chain of references from the current scope leading to it. If an object is held in scope, but never gets used, we say there is a memory leak. Such leaks are especially problematic when they occur periodically, as the memory allocated by the virtual machine increases over time, eventually breaking things.

By setting a key-value pair in a weap map, no reference to the property *key* is created. Instead, references to keys which are internal to weak maps can be thought as "weak", meaning that from the point of view of the garbage collector they are ignored. In particular, property keys of a weak map cannot be enumerated. Also, weak maps do not have a `size` property analogous to maps as this would expose garbage collector behaviour which should be kept hidden.

```javascript
let weakMap = WeakMap();

// This is effectively a noop, the key-value pair can immediately be garbage collected
weakMap.set({}, 'noise');
```

Weak maps allows for "meta-data" to be associated to an object without obstructing garbage collection would the object otherwise escape the current scope.

```javascript
// Expose user metadata to the rest of the program
let userMetadata = WeakMap();

server.on('userConnect', function (user) {
  userMetadata.set(user, { connectionTime: Date.now() });

  server.on('userDisconnect', function () {
    // Do stuff with `user` and discard of it, automatically discarding `userMetadata`
  });
});
```

Because literals in JavaScript are passed by *value* (instead of by *reference*), literals cannot be weak map keys.

```javascript
let weakMap = WeakMap();

// TypeError: Invalid value used as weak map key
weakMap.set('example string literal', 0);
```

From the point of view of JavaScript code, there is no way to confirm that objects have been garbage collected. One has to have faith that the virtual machine is doing its job properly. Alternatively, the guts of the virtual machine can be inspected with a debugger, such as [node-inspector](https://github.com/dannycoates/node-inspector).

Symbols
---

Symbols extend the set of key values, i.e. object property identifiers. In ES5 the set of key values corresponds precisely to the set of strings.

```javascript
let a = {};
let debugSymbol = Symbol();

a[debugSymbol] = 'This property value is identified by a symbol';
```

Symbols are unique by construction which make them useful for avoiding name collisions. By unique, we mean that two symbols created at different points in time will never refer to the same property of an object.

In ES5, name collisions avoidance is hacked by using names which are *unlikely* to collide. The following snippet from jQuery's source illustrates this.

```javascript
jQuery.extend({
  // Unique for each copy of jQuery on the page
  // Non-digits removed to match rinlinejQuery
  expando: "jQuery" + ( core_version + Math.random() ).replace( /\D/g, "" ),
```

Uniqueness has interesting consequences.

```javascript
let a = Map();
a.set(Symbol(), 'Noise');

// Prints "1" although the one element cannot be accessed!
console.log(a.size);
```

For informational or debugging purposes, an immutable name can be given when instantiating a symbol.

```javascript
let testSymbol = Symbol('This is a test');

// Prints "This is a test"
console.log(testSymbol.name);
```

TODO: Discuss private symbols when the get implemented.

Object observation
---

TODO

Typed arrays and array buffers
---

TODO

Concluding note
---

As of Node v0.11.3, a lot of the ES6 features providing new semantics to the language are available. As we saw, these include proxies, symbols and weak maps. We look forward for tasty syntax sugar yet to come, such as rest and spread operators, destructuring and classes. Yum yum.
