<!-- https://github.com/ssshooter/javascript-tutorial-cn/tree/master/1-js/06-advanced-functions/03-closure -->
# Closure

JavaScript 是一个很 function-oriented 的语言。他带来了很大的操作自由。一个函数创建一次，可以拷贝到另一个变量，或者作为一个参数传入另一个函数然后在一个全新的环境调用。

我们知道函数可以访问它外部的变量，这个 feature 非常常用。

但是当外部变量改变时会发生什么？函数时获取最新的值，还是函数创建当时的值？

还有一个问题，当函数被送到其他地方再调用……他能访问那个地方的外部变量吗？

不同语言的表现有所不同，下面我们研究一下 JavaScript 中的表现。

[cut]

## 两个问题

我们先思考下面两种情况，再慢慢学习内部机制，之后你就可以回答以下的问题，甚至解决未来遇到的更复杂的问题。

1. `sayHi` 函数使用了外部变量 `name`。函数运行时，会使用两个值中的哪个？

    ```js
    let name = "John";

    function sayHi() {
      alert("Hi, " + name);
    }

    name = "Pete";

    *!*
    sayHi(); // "John" 还是 "Pete"？
    */!*
    ```

    这个情况不论是浏览器端还是服务器端都很常见。函数很可能在它创建一段时间后才执行，例如等待用户操作或者网络请求。

    问题是：函数是否会选择变量最新的值呢？


2. `makeWorker` 函数创造并返回了另一个函数。这个新函数可以在任何地方调用。他会访问创建时的变量还是调用时的变量呢？

    ```js
    function makeWorker() {
      let name = "Pete";

      return function() {
        alert(name);
      };
    }

    let name = "John";

    // 创建函数
    let work = makeWorker();

    // 调用函数
    *!*
    work(); // "Pete" （创建时） 还是 "John" （调用时）？
    */!*
    ```


## Lexical Environment （词法环境）

要理解里面发生了什么，必须先明白“变量”到底是什么。

在 JavaScript 里，任何运行中函数、代码块、整个 script 都会关联一个被叫做 *Lexical Environment （词法环境）* 的对象。

Lexical Environment 对象包含两个部分：

1. *Environment Record （环境记录）*是一个拥有全部局部变量作为属性的对象（以及其他如 `this` 值的信息）。
2. *outer lexical environment （外部词法环境）*的引用，通常词法关联外面一层代码（花括号外一层）。 

所以，“变量”就是内部对象 Environment Record 的一个属性。要改变一个对象，意味着改变 Lexical Environment 的属性。

例如在这段简单的代码中，只有一个 Lexical Environment：

![lexical environment](lexical-environment-global.png)

这就是所谓 global Lexical Environment （全局语法环境），关联于整个 script。对于浏览端，整个 `<script>` 标签共享一个全局环境。

上图中，正方形代表 Environment Record （变量储存），箭头代表 outer reference （外部引用）。global Lexical Environment 没有外部引用，所以指向 `null`。

下图展示 `let` 变量的工作机制：

![lexical environment](lexical-environment-global-2.png)

右边的正方形描述 global Lexical Environment 在执行中如何改变：

1. 脚本开始运行，Lexical Environment 空。
2. `let phrase` 定义出现了。因为没有赋值所以储存为 `undefined` 。
3. `phrase` 被赋值。
4. `phrase` 被赋新值。

看起来很简单对不对？

总结：

- 变量是一个特殊内部对象的属性，关联于执行时的块、函数、 script 。
- 对变量的操作实际上是对这个对象属性的操作。

### Function Declaration （函数声明）

Function Declarations 是特别的。不同于 `let` 变量， they are processed not when the execution reaches them, but when a Lexical Environment is created. 对于 global Lexical Environment ，这意味着 script 开始运行的时候。

这就是函数可以在定义前调用的原因。That is why we can call a function declaration before it is defined.

下面代码描述 Lexical Environment 开始非空。因为有 `say` 函数声明，之后又有了 `let` 声明的 `phrase` ：
The code below demonstrates that the Lexical Environment is non-empty from the beginning. It has `say`, because that's a Function Declaration. And later it gets `phrase`, declared with `let`:

![lexical environment](lexical-environment-global-3.png)


### Inner and outer Lexical Environment （内部词法环境和外部词法环境）

调用 `say()` 的过程中，它使用了外部变量，一起看看这里面发生了什么。
During the call, `say()` uses an outer variable, so let's look at the details of what's going on.

首先，函数运行，一个新的函数 Lexical Environment 被自动创建。这是所有函数的通用规则。这个新的 Lexical Environment 用于当前运行函数的存放局部变量和形参。
First, when a function runs, a new function Lexical Environment is created automatically. That's a general rule for all functions. That Lexical Environment is used to store local variables and parameters of the call.

<!--
    ```js
    let phrase = "Hello";
    
    function say(name) {
     alert( `${phrase}, ${name}` );
    }
    
    say("John"); // Hello, John
    ```-->

执行 `say("John")` 时的 Lexical Environment （被箭头标记）：
Here's the picture of Lexical Environments when the execution is inside `say("John")`, at the line labelled with an arrow:

![lexical environment](lexical-environment-simple.png)

函数调用时我们看到两个 Lexical Environment ：里面一个是函数调用产生的，外面的是全局的：
During the function call we have two Lexical Environments: the inner one (for the function call) and the outer one (global):

- 内层 Lexical Environment 关联于当前执行的 `say` 。它只有一个变量： `name` ，函数的实参。我们调用 `say("John")` ，所以 `name` 的值是 `"John"` 。
- 外层 Lexical Environment 是 global Lexical Environment 。

内层 Lexical Environment 有一个 `outer` 属性，指向外层 Lexical Environment。

**代码要访问一个变量，首先搜索内层 Lexical Environment ，接着是外层，再外层，直到链的结束。**

如果走完整条链变量都找不到，在 strict mode 就会报错了。不使用 `use strict` 的情况下，对未定义变量的赋值，会创造一个新的全局变量。

下面一起看看变量搜索如何处理：
Let's see how the search proceeds in our example:

- `say` 里的 `alert` 想要访问 `name` ，立即就能在当前函数的 Lexical Environment 找到。
- 对于 `phrase` ，局部变量不存在 `phrase` ，所以要循着 `outer` 在全局变量里找到。

![lexical environment lookup](lexical-environment-simple-lookup.png)

现在我们可以回答本章开头的第一个问题了。

**A function gets outer variables as they are now; it uses the most recent values.**

这是一种 described mechanism 。旧变量值不储存在任何地方，函数需要他们的时候，它得到来源于自身或外部 Lexical Environment 的当前值。
That's because of the described mechanism. Old variable values are not saved anywhere. When a function wants them, it takes the current values from its own or an outer Lexical Environment.

所以第一个问题的答案是 `Pete` ：

```js run
let name = "John";

function sayHi() {
  alert("Hi, " + name);
}

name = "Pete"; // (*)

*!*
sayHi(); // Pete
*/!*
```


上述代码的执行流：

1. global Lexical Environment 存在 `name: "John"` 。
2. `(*)` 行中，全局变量修改了，现在成了这样 `name: "Pete"` 。
3. `say()` 执行的时候， 取外部 `name` 。此时在 global Lexical Environment 中已经是 `"Pete"`。


```smart header="一次调用 -- 一个 Lexical Environment"
请注意，每当一个函数运行，就会创建一个新的 function Lexical Environment。

如果一个函数被多次调用，那么每次调用都会生成一个属于当前调用的新的 Lexical Environment ，里面装载着当前调用的变量和实参。
```

```smart header="Lexical Environment 是一个标准对象 （specification object）"
"Lexical Environment" 是一个标准对象 （specification object）。我们不能直接获取或设置它，JavaScript 引擎也可能优化它，抛弃未使用的变量来节省内存或者作其他优化，但是可见行为应该如上面所述。
```


## 嵌套函数

一个函数创造于另一个函数中，称为“嵌套”。

这在 JavaScript 很容易做到。

我们可以这么组织代码：

```js
function sayHiBye(firstName, lastName) {

  // helper nested function to use below
  function getFullName() {
    return firstName + " " + lastName;
  }

  alert( "Hello, " + getFullName() );
  alert( "Bye, " + getFullName() );

}
```

这个*嵌套函数* `getFullName()` 给我们带来便利。它能访问访问外部变量，返回全称。

更有趣的是，嵌套函数可以被 return ，作为一个新对象的属性（）或者作为自己的结果。这样它们就能在其他地方使用。无论在哪里，它都会访问同样的外部变量。
What's more interesting, a nested function can be returned: either as a property of a new object (if the outer function creates an object with methods) or as a result by itself. It can then be used somewhere else. No matter where, it still has access to the same outer variables.

一个构造函数（see the chapter <info:constructor-new>）的例子：

```js run
// 构造函数返回一个新对象
function User(name) {

  // 嵌套函数创造对象方法
  this.sayHi = function() {
    alert(name);
  };
}

let user = new User("John");
user.sayHi(); // 方法返回外部 "name"
```

一个 return 函数的例子：

```js run
function makeCounter() {
  let count = 0;

  return function() {
    return count++; // has access to the outer counter
  };
}

let counter = makeCounter();

alert( counter() ); // 0
alert( counter() ); // 1
alert( counter() ); // 2
```

我们接着研究 `makeCounter` 。counter 函数每调用一次就会返回下一个数。尽管这很简单，但只要轻微修改，它便具有一定的实用性，例如[伪随机数生成器](https://en.wikipedia.org/wiki/Pseudorandom_number_generator)。

counter 内部如何工作？

内部函数运行， `count++` 中的变量由内到外搜索：

![](lexical-search-order.png)

1. 嵌套函数局部变量……
2. 外层函数……
3. 直到全局变量。

第二步我们找到了 `count` 。当外部变量被修改，它所在的地方就被修改。所以 `count++` 检索外部变量并对其加一是操作于该变量自己的 Lexical Environment 。就像操作了 `let count = 1` 一样。

这里需要思考两个问题：

1. 我们能通过 `makeCounter` 以外的方法重置 `counter` 吗？
2. 如果我们可以多次调用 `makeCounter()` ，返回了很多 `counter` 函数，他们的 `count` 是独立的还是共享的？

继续阅读前可以先尝试思考一下。

...

ok ？

那我们开始揭晓谜底：

1. 没门。 `counter` 是局部变量，不可能在外部直接访问。
2. 每次调用 `makeCounter()` 都会新建 Lexical Environment，每一个环境都有自己的 `counter` 。所以不同 counter 里的 `count` 是独立的。

一个 demo ：

```js run
function makeCounter() {
  let count = 0;
  return function() {
    return count++;
  };
}

let counter1 = makeCounter();
let counter2 = makeCounter();

alert( counter1() ); // 0
alert( counter1() ); // 1

alert( counter2() ); // 0 （独立）
```


Hopefully, the situation with outer variables is quite clear for you now. But in more complex situations a deeper understanding of internals may be required. So let's dive deeper.

## Environment 细节

现在我们对 closure （闭包）有了初步了解，可以开始深入细节了。

下面是 `makeCounter` 例子的动作分解，跟着看你就能理解一切了。注意， `[[Environment]]` 属性我们之前还未介绍。

1. 脚本开始运行，此时只存在 global Lexical Environment ：

    ![](lexenv-nested-makecounter-1.png)

    这时候只有 `makeCounter` 一个函数，这是函数声明，还未被调用。

    所有函数都带着一个隐藏属性 `[[Environment]]` “诞生”。 `[[Environment]]` 指向它们创建的 Lexical Environment 。是`[[Environment]]` 让函数知道它“诞生”于什么环境。

    `makeCounter` 创建于 global Lexical Environment ，所以 `[[Environment]]` 指向它。

    换句话说，Lexical Environment 在函数诞生时就“铭刻”在这个函数中。`[[Environment]]` 是指向 Lexical Environment 的隐藏函数属性。

2. 代码继续走， `makeCounter()` 登上舞台。这是代码运行到 `makeCounter()` 瞬间的快照：

    ![](lexenv-nested-makecounter-2.png)

    `makeCounter()` 调用时，保存当前变量和实参的 Lexical Environment 已经被创建。

    Lexical Environment 储存两个东西：
    1. 带有局部变量的 Environment Record 。In our case `count` is the only local variable (appearing when the line with `let count` is executed).
    2. 外部词法引用，被绑定到函数 `[[Environment]]` 。例子里 `makeCounter` 的 `[[Environment]]` 引用了 global Lexical Environment 。

    所以这里有两个 Lexical Environments ：全局，和 `makeCounter` （outer 引用全局）。

3. 在 `makeCounter()` 执行的过程中，创建了一个嵌套函数。

    这无关于函数创建使用的是 Function Declaration （函数声明）还是 Function Expression （函数表达式）。所有函数都会得到引用他们被创建时 Lexical Environment 的 `[[Environment]]` 属性。

    这个嵌套函数的 `[[Environment]]` 是 `makeCounter()` （它的诞生地）的 Lexical Environment：

    ![](lexenv-nested-makecounter-3.png)

    同样注意，这一步是函数声明而非调用。

4. 代码继续执行，`makeCounter()` 调用结束，内嵌函数被赋值到全局变量 `counter` ：

    ![](lexenv-nested-makecounter-4.png)

    这个函数只有一行： `return count++` 。

5. `counter()` 被调用，自动创建一个 “空” Lexical Environment 。 此函数无局部变量，但是 `[[Environment]]` 引用了外面一层，所以它可以访问 `makeCounter()` 的变量。

    ![](lexenv-nested-makecounter-5.png)

    要访问变量，先检索自己的 Lexical Environment (empty)，然后是 `makeCounter()` 的，最后是全局的。例子中在最近的外层 Lexical Environment `makeCounter` 中发现了 `count` 。

    重点来了，内存在这里是怎么管理的？尽管 `makeCounter()` 调用结束了，它的 Lexical Environment 依然保存在内存中，这是因为嵌套函数的 `[[Environment]]` 引用了它。

    通常， Lexical Environment 对象随着使用它的函数的存在而存在。没有函数引用它的时候，它便会被清除。

6. `counter()` 函数不只是返回 `count` ，还会对其 +1 操作。这个修改已经在“适当的位置”完成了。`count` 的值在它被找到的环境中被修改。

    ![](lexenv-nested-makecounter-6.png)

    这一步出了返回了新的 `count` ，其他完全相同。

7. 下一个 `counter()` 调用操作同上。

本章开头第二个问题的答案现在显而易见了。

以下代码的 `work()` 函数通过外层 lexical environment 引用了它原地点的 `name` ：

![](lexenv-nested-work.png)

所以这里的答案是 `"Pete"` 。

但是如果 `makeWorker()` 没了 `let name` ，如我们所见，作用域搜索会到达外层，获取全局变量。这个情况下答案会是 `"John"` 。

```smart header="闭包 （Closure）"
开发者们都应该知道编程领域的通用名词闭包 （closure）。

[Closure](https://en.wikipedia.org/wiki/Closure_(computer_programming)) 是一个记录并可访问外层变量的函数。在一些编程语言中，这是不可能的，或者要以一种特殊的方式书写以实现这个功能。但是如上面解释的， JavaScript 的所有函数都（很自然地）是个闭包。（有一个例外，详见<info:new-function>）

这就是闭包：它们使用 `[[Environment]]` 属性自动记录各自的创建地点，然后由此访问外部变量。

在前端面试中，如果面试官问你什么是闭包，正确答案应该包括闭包的定义，以及解释为何 JavaScript 的所有函数都是闭包，最好可以再简单说说里面的技术细节： `[[Environment]]` 属性和 Lexical Environments 的原理。
```

## 代码块、循环、 IIFE

上面的例子都着重于函数，但是 Lexical Environment 也存在于代码块 `{...}` 。

它们在代码块运行时创建，包含块局部变量。这里有一些例子。

## If

In the example below, when the execution goes into `if` block, the new "if-only" Lexical Environment is created for it:

<!--
    ```js run
    let phrase = "Hello";
    
    if (true) {
        let user = "John";

        alert(`${phrase}, ${user}`); // Hello, John
    }

    alert(user); // Error, can't see such variable!
    ```-->

![](lexenv-if.png)

The new Lexical Environment gets the enclosing one as the outer reference, so `phrase` can be found. But all variables and Function Expressions declared inside `if` reside in that Lexical Environment and can't be seen from the outside.

For instance, after `if` finishes, the `alert` below won't see the `user`, hence the error.

## For, while

For a loop, every iteration has a separate Lexical Environment. If a variable is declared in `for`, then it's also local to that Lexical Environment:

```js run
for (let i = 0; i < 10; i++) {
  // Each loop has its own Lexical Environment
  // {i: value}
}

alert(i); // Error, no such variable
```

That's actually an exception, because `let i` is visually outside of `{...}`. But in fact each run of the loop has its own Lexical Environment with the current `i` in it.

After the loop, `i` is not visible.

### Code blocks

We also can use a "bare" code block `{…}` to isolate variables into a "local scope".

For instance, in a web browser all scripts share the same global area. So if we create a global variable in one script, it becomes available to others. But that becomes a source of conflicts if two scripts use the same variable name and overwrite each other.

That may happen if the variable name is a widespread word, and script authors are unaware of each other.

If we'd like to avoid that, we can use a code block to isolate the whole script or a part of it:

```js run
{
  // do some job with local variables that should not be seen outside

  let message = "Hello";

  alert(message); // Hello
}

alert(message); // Error: message is not defined
```

The code outside of the block (or inside another script) doesn't see variables inside the block, because the block has its own Lexical Environment.

### IIFE

In old scripts, one can find so-called "immediately-invoked function expressions" (abbreviated as IIFE) used for this purpose.

They look like this:

```js run
(function() {

  let message = "Hello";

  alert(message); // Hello

})();
```

Here a Function Expression is created and immediately called. So the code executes right away and has its own private variables.

The Function Expression is wrapped with brackets `(function {...})`, because when JavaScript meets `"function"` in the main code flow, it understands it as the start of a Function Declaration. But a Function Declaration must have a name, so there will be an error:

```js run
// Error: Unexpected token (
function() { // <-- JavaScript cannot find function name, meets ( and gives error

  let message = "Hello";

  alert(message); // Hello

}();
```

We can say "okay, let it be so Function Declaration, let's add a name", but it won't work. JavaScript does not allow Function Declarations to be called immediately:

```js run
// syntax error because of brackets below
function go() {

}(); // <-- can't call Function Declaration immediately
```

So, round brackets are needed to show JavaScript that the function is created in the context of another expression, and hence it's a Function Expression. It needs no name and can be called immediately.

There are other ways to tell JavaScript that we mean Function Expression:

```js run
// 创建 IIFE 的方法

(function() {
  alert("Brackets around the function");
}*!*)*/!*();

(function() {
  alert("Brackets around the whole thing");
}()*!*)*/!*;

*!*!*/!*function() {
  alert("Bitwise NOT operator starts the expression");
}();

*!*+*/!*function() {
  alert("Unary plus starts the expression");
}();
```

In all the above cases we declare a Function Expression and run it immediately.

## Garbage collection

Lexical Environment objects that we've been talking about are subject to the same memory management rules as regular values.

- Usually, Lexical Environment is cleaned up after the function run. For instance:

    ```js
    function f() {
      let value1 = 123;
      let value2 = 456;
    }

    f();
    ```

    Here two values are technically the properties of the Lexical Environment. But after `f()` finishes that Lexical Environment becomes unreachable, so it's deleted from the memory.

- ...But if there's a nested function that is still reachable after the end of `f`, then its `[[Environment]]` reference keeps the outer lexical environment alive as well:

    ```js
    function f() {
      let value = 123;

      function g() { alert(value); }

    *!*
      return g;
    */!*
    }

    let g = f(); // g is reachable, and keeps the outer lexical environment in memory
    ```

- Please note that if `f()` is called many times, and resulting functions are saved, then the corresponding Lexical Environment objects will also be retained in memory. All 3 of them in the code below:

    ```js
    function f() {
      let value = Math.random();

      return function() { alert(value); };
    }

    // 3 functions in array, every one of them links to Lexical Environment
    // from the corresponding f() run
    //         LE   LE   LE
    let arr = [f(), f(), f()];
    ```

- A Lexical Environment object dies when it becomes unreachable: when no nested functions remain that reference it. In the code below, after `g` becomes unreachable, the `value` is also cleaned from memory;

    ```js
    function f() {
      let value = 123;

      function g() { alert(value); }

      return g;
    }

    let g = f(); // while g is alive
    // there corresponding Lexical Environment lives

    g = null; // ...and now the memory is cleaned up
    ```

### Real-life optimizations

As we've seen, in theory while a function is alive, all outer variables are also retained.

But in practice, JavaScript engines try to optimize that. They analyze variable usage and if it's easy to see that an outer variable is not used -- it is removed.

**An important side effect in V8 (Chrome, Opera) is that such variable will become unavailable in debugging.**

Try running the example below in Chrome with the Developer Tools open.

When it pauses, in the console type `alert(value)`.

```js run
function f() {
  let value = Math.random();

  function g() {
    debugger; // in console: type alert( value ); No such variable!
  }

  return g;
}

let g = f();
g();
```

As you could see -- there is no such variable! In theory, it should be accessible, but the engine optimized it out.

That may lead to funny (if not such time-consuming) debugging issues. One of them -- we can see a same-named outer variable instead of the expected one:

```js run global
let value = "Surprise!";

function f() {
  let value = "the closest value";

  function g() {
    debugger; // in console: type alert( value ); Surprise!
  }

  return g;
}

let g = f();
g();
```

```warn header="再会！"
This feature of V8 is good to know. If you are debugging with Chrome/Opera, sooner or later you will meet it.

That is not a bug in the debugger, but rather a special feature of V8. Perhaps it will be changed sometime.
You always can check for it by running the examples on this page.
```