Node.JS 对 ECMAScript 6 的支持
===

[译者] 本文是对 [ECMAScript 6 in Node.JS](https://github.com/JustinDrake/node-es6-examples) 的中文翻译。为了避免概念混淆，在翻译到 ECMAScript 6  的专有特性名称的时候，我还是保留英文原文，因为我不能确定这些特性最终的权威中文译名是什么。

ECMAScript 6 (简写为 ES6) 的特性已经被 Node 原生支持，本文对这些特性进行说明，并提供样例。执行这些样例无需转换程序。我们希望读者能从这些 ES6 的子集中找到感兴趣的内容。

对于 ES6 的底层哲学和大方向的目标目前已经取得基本一致，但是实现细节在最终规范发布之前还是会有所修改。Node 中对 ES6 的实验性的实现未必完全遵循最新的草案([HTML 格式的草案](http://people.mozilla.org/~jorendorff/es6-draft.html))。

使用 `node --v8-options | grep harmony` 命令可以获得目前 Node 实现的 ES6 特性的列表。Node 非稳定分支 0.11x 对 ES6 的支持要比稳定分支 0.10.x 好很多。简单起见，我们假设你用的是 0.11.x 分支，不过大部分例子在 0.10.x 下也能工作。在不同的 Node 版本间切换，我们建议你使用 [n](https://github.com/visionmedia/n) 这个出色的 Node 版本管理工具。

使用单一的命令行选项 `--harmony` 将打开大部分 ES6 实验特性。不过在 v0.11.3 版本，你仍然需要 `--use_strict` 选项来运行 block scoping 例子，用 `--harmony_generators` 选项来运行 generator 例子。

欢迎 Pull requests。阅读愉快:)

Block scoping
---

让我们从 `let` 开始。你可以把 `let` 理解为代码块范围的 `var` 变量声明。

```javascript
{ let a = 'I am declared inside an anonymous block'; }
console.log(a); // ReferenceError: a is not defined
```

在 ES6 之前，JavaScript 只有函数范围的变量作用域。这被认为是一个设计缺陷，开发者们也在尝试绕过这个问题。我们用以下两个例子来说明 ES6 的改进。

第一个例子是在 ES5 中实现私有变量（利用函数闭包）：

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

第二个例子来处理变量 “提升”（hoisting）问题。

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

最后一个例子的 `for` 循环产生了一个隐式的块作用域，这个块作用域包含一个 `i` 变量声明；同时随着每次循环，也会产生对应该循环的新的块作用域。

ES6 的设计者也为 `let` 构思出了一个小姐妹：关键字 `const`，用作块作用域的 *常数* 变量声明。

```javascript
const a = 'You shall remain constant!';

// SyntaxError: Assignment to constant variable
a = 'I wanna be free!';
```

最后，ES6 也修复了一个关于函数声明的长期存在的问题。下面的例子的行为在 ES5 中并没有准确定义：

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

在 `if` 块中的 `f` 重声明是否应该被提升，或者由于条件为 `false` 而被跳过? 是否应该把新的 `f` 声明限定在 `if` 块内? 不同的浏览器用不同的逻辑来处理这个问题。在 ES6 中，函数声明是拥有其块作用域的，因此上面的例子会打印出 `I am outside!`。

在ES6中，在相同的块作用域中对一个变量多次使用 `let` 声明，将会导致跑出语法错误异常。在函数范围内多次使用 `var` 声明将不会抛出类似的异常，这可能会对开发者造成一定的疑惑。

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

Generator 允许类似函数的一段代码能够被分割成“一片一片”地执行。当“一片”执行完毕，这段代码的执行被暂停，直到“下一片”开始才继续执行。Generators 的语法类似函数，但是使用 `function*` 关键字而不是 `function`。代码执行的流控制是通过 `yield` 语句来控制的。

```javascript
function* argumentsGenerator() {
  for (let i = 0; i < arguments.length; i += 1) {
    yield arguments[i];
  }
}
```

（请注意，虽然 `yield` 关键字 *不是* ES5的保留字，新的 `function*` 语法也确保原来的 ES5 的程序如果使用 "yield" 来作为变量名的话，在迁移到 ES6 之后不会出现执行行为的变化 ）。

Generators 如此有用的原因在于，其返回（产生）一个迭代器 (iterators) 对象。迭代器对象包含 `next` 方法，每次调用该方法将执行 generator  中的代码。`next` 方法被重复调用，每次调用将在 generator 执行一段代码，直到遇到 `yield` 关键字，从而暂停 generator 的代码执行，返回给调用者 。

```javascript
var argumentsIterator = argumentsGenerator('a', 'b', 'c');

// Prints "a b c"
console.log(
    argumentsIterator.next().value,
    argumentsIterator.next().value,
    argumentsIterator.next().value
);
```

`next` 方法返回一个对象，该对象包含 `value` 和 `done` 两个属性。`done` 属性用来表明 generator 的所有工作是否都已经执行完了（无需再次调用 `next` 方法）。`value` 属性代表了 `yield` 或者 generator 的返回值。在 generator 完全返回之前，`done` 属性将一直返回 `false`。如果在 `done` 属性为 `true` 之后继续调用 `next` 方法，那么将抛出一个异常。

与 generators 和 iterators 相伴, ES6 也为迭代实现提供了语法糖（下例中的 `of`）：

```javascript
// Prints "a", "b", "c"
for(let value of argumentsIterator) {
  console.log(value);
}
```

Generator 是实现不确定迭代长度的需求的理想工具。

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

...可以优雅地枚举如下：

```javascript
// Prints the Fibonacci numbers less than 1000
for(let value of fibonacci(1000)) {
  console.log(value);
}
```

Generator 也可以用来替代传统的回调式流程控制，用来避免`回调金字塔`和 `回调地狱`。以下两个库，[task.js](https://github.com/mozilla/task.js) 和 [gen-run](https://github.com/creationix/gen-run) 用来帮助开发者以顺序风格来编写异步 JavaScript 代码。

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

顺序流程控制也允许使用 `try`-`catch` 语句。在 Node 库中广泛使用的在回调链中返回错误对象的方法所带来的负担，在 ES6 将被大大缓解。

最后，我们也可以在一个 generator 中通过 `yield*` 来让另一个 "委派的 `yield` 来产生 `yield` 的结果。

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

一个 Proxy 是一个“元编程”对象，该对象将其他原始对象的行为替换为新的函数调用，这个新的函数称为 trap。Trap 是关联到某个句柄对象的方法。

```javascript
var random = Proxy.create({
  get: function () {
    return Math.random();
  }
});
```

`Proxy.create` 创建了一个 proxy，该 proxy 的句柄对象j就是是传递给它的第一个参数 (`{get: ...}`)。在这个例子中，我们又一个单一的 `get` trap，重载了原本的属性读操作。每次对 `random` proxy 对象的读取，都将在运行时计算出一个新的随机数。

```javascript
// Prints three random numbers
console.log(random.value, random.value, random.value);
```

类似地，属性写可以被 `set` trap 所覆盖。

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

接下来的例子演示了如何通过 Proxy 让一个新对象工作起来就像 JavaScript 的原生对象，该例子实现了 Python 风格的数组，即支持负的数组下标。

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

如上例：下标 `-1` 指向数组的最后一个元素，`-2` 指向倒数第二个，以此类推。

注意，`set` 有三个参数：`receiver` 指向 proxy 本身，`index` 是属性名称，`value` 则为属性值。

```javascript
// Prints "gamma"
console.log(pythonArray(['alpha', 'beta', 'gamma'])[-1]);
```

Proxy 也可被用来实现精炼的数据绑定。以 Backbone.JS models 为例，数据绑定是通过调用“昂贵”的 `model.get` 和 `model.set` 方法来实现的。使用 proxy 则可以避免使用这种不直观的语法。

让我们以一个复杂一些的安全用例来为 proxy 结尾，请带上你“想象”的帽子。

假设一个函数 `f` 共享一个对象 `o` 给另一个函数 `g`；之后 `f` 又想撤回这个共享。为了实现这个逻辑：`f` 可以给 `g` 一个 封装了对 `o` 访问的代理对象 `p` 。`p` 的 trap 能否被 `g` 调用则依赖 `f ` 的私有属性 `k` 的状态，这样 'f' 就可以通过 `k` 来控制 `p` 能否为其他 `p` 的拥有者提供服务。事实上，我们还可以在 `p` 的句柄对象 `h` 中添加一个代理对象 `q` ，在一个 `p` 的 `get` trap中来实现对 `p` 的所有 traps 的访问控制。

整理一下：`g` 通过 `p` 访问 `o`。`g` 对 `p` 的 trap 的调用，将会触发 `p` 的句柄 'h' 中的代理对象 `q` 的 `get` trap，`get` trap 实现了通过 `f` 的私有变量 `k` 进行访问控制的目标。[译者] 简单说，就是用一个 `q` 的 `get` trap 控制所有的 `p` 的 traps 的访问。 这里确实需要例子才能说明清楚，否则读者很容易被 trapped :)

TODO: Add examples which are not `get` or `set`.

Maps 和 sets
---

Map 可以被理解为类似 ES5 的 Object ，但是其属性的 key 可以是任何类型的 Object ，而不仅仅是字符串。在 ES5 中，当对 Object 属性进行访问时，其 key 的 `toString` 方法会被隐含调用以获得 key 的字符串来作为对象属性的标识符。这种方法对于 key 值为`{}` 的情况则没有什么意义，因为 `({}.toString())` 的字符串值总是为 `[object Object]`。

参见下面 Map 的例子：

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

Set 是一种数据结构，包含有限的元素，并且每个元素仅仅出现一次。其构造器是 `Set`，有一组简单的 API。

```javascript
// Prints [ 'constructor', 'size', 'add', 'has', 'delete', 'clear' ]
console.log(Object.getOwnPropertyNames(Set.prototype));
```

举例来说，我们做一个用户调研，题目是“什么是他们生活中最愉快的六件事”：

```javascript
let surveyAnswers = ['sex', 'sleep', 'sex', 'sun', 'sex', 'cinema'];

let pleasures = Set();
surveyAnswers.forEach(function (pleasure) {
  pleasures.add(pleasure);
});

// Prints the number of pleasures in the survey, not counting duplicates
console.log(pleasures.size);
```

遗憾的是，将 Array 转换到 Set ，或者对 Set 进行迭代枚举等功能尚不为 Node.JS 所支持。

Map 和 Set 数据结构都可以通过对既有的 ES5 Object 修改而实现。我们下面要讨论的 Weak Map 则 *不可能* 在 ES5 中模拟出来。

Weak maps
---

Weak Map 适合一些 Map 和 Set 完全不同的场景，因此不仅仅是语法糖。Weak Map 和 Map 类似，但是它是和垃圾回收密切相关的，提供了工具来帮助你写出无内存泄露的程序。

JavaScript 的虚拟机，例如 Node 用到的 V8，周期性地清理那些不在作用域中的 Objects 所占用的内存。判断一个 Object 是否在作用域中被使用，是看当前作用域是否还有引用链指向它。如果一个 Object 存在于作用域，但是却再也不能被使用，那么我们就认为这是一个内存泄露。类似的泄露对象如果随着时间的推移被虚拟机不断地分配出来的话，最终会由于内存耗尽而造成不可预期的后果。

在 Weap Map 中设定 key-value 对的时候，并没有建立到属性 *key* 对象的真正引用。我们可以认为在 Weak Map 中，到其属性 *key* 对象的引用是一种 “弱引用”。从垃圾回收器的观点来说，在对 *key* 对象进行回收时，从 Weak Map来得引用是被完全忽略的。特别要指出的是，Weak Map 的属性 keys 是不能被枚举的；并且，Weak Map 也没有 Map 中的 `size` 属性，因为这会暴露垃圾回收器的行为，而这是不应该对外暴露的。

```javascript
let weakMap = WeakMap();

// This is effectively a noop, the key-value pair can immediately be garbage collected
weakMap.set({}, 'noise');
```

Weak Map 可以为某个 object 添加 “元数据”，而这个“元数据”不会阻碍垃圾回收器对该 object 的回收。

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

在 JavaScript 中，由于字面量是按照 *值* （而不是以 *引用* ）方式传递的，字面量不能作为 Weak Map 的 key。

```javascript
let weakMap = WeakMap();

// TypeError: Invalid value used as weak map key
weakMap.set('example string literal', 0);
```

从 JavaScript 的观点来看，没有办法知道一个对象是否被垃圾回收了。我们必须信任虚拟机能够正确地做好这些事。当然，虚拟机的内部情况是可以通过调试器来查看的，例如：[node-inspector](https://github.com/dannycoates/node-inspector)。

Symbols
---

Symbol 是对 key 的取值类型的扩展，也就是说是对对象属性标示符类型的扩展。在 ES5 中，对象属性标识符（ key 的值类型）只有字符串一种。

```javascript
let a = {};
let debugSymbol = Symbol();

a[debugSymbol] = 'This property value is identified by a symbol';
```

`Symbol()` 每次返回唯一的新的 Symbol 来避免命名冲突。唯一的意思是，在不同时间点产生的两个 Symbols 不会被指向对象的同一个属性。

在 ES5 中，避免名字冲突的方法是使用一种 *不容易* 冲突的命名算法，例如来自于 jQuery 的源代码片段：

```javascript
jQuery.extend({
  // Unique for each copy of jQuery on the page
  // Non-digits removed to match rinlinejQuery
  expando: "jQuery" + ( core_version + Math.random() ).replace( /\D/g, "" ),
```

唯一性会导致有趣的结果：
Uniqueness has interesting consequences.

```javascript
let a = Map();
a.set(Symbol(), 'Noise');

// Prints "1" although the one element cannot be accessed!
console.log(a.size);
```

为了便于代码理解或调试方便，创建一个 Symbol 的时候可以带一个不可改变的名字。

```javascript
let testSymbol = Symbol('This is a test');

// Prints "This is a test"
console.log(testSymbol.name);
```

TODO: 在 V8 实现后增加对 Private Symbol 的讨论。

Object observation
---

TODO

Typed arrays 和 array buffers
---

TODO

结论
---

截止到 Node v0.11.3, 大量的 ED6 新语法特性被支持。如上所述，包含 Proxy、Symbols 和 Weak Maps。我们期待着品尝到更多的“语法糖”，例如 rest 和 spread 操作、destructuring和 class... Yum yum.