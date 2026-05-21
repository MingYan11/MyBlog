---
title: 前端面试题-JavaScript
date: 2026-05-21 15:00:00
tags: [前端, 面试, JavaScript]
categories: 前端面试
---

### 数据类型

- 基本数据类型：
  - String
  - Number
  - Boolean
  - Null
  - Undefined
  - Symbol (ES6)
    BigInt (ES2020)
- 引用数据类型：主要是: Object (包括普通对象、Array、Function、Date等)。
  - 特点：
    - 值本身（即对象实体）存储在堆内存 (Heap)中，而变量的值是指向该堆内存地址的引用，该引用存储在栈内存中；
    - 变量之间的赋值是引用的复制。

**null和undefined区别**

- null：表示“无值”或“空值”，是一个对象类型的值，通常用于表示变量应该有一个对象，但目前为空。
- undefined：表示变量已声明但未初始化，或者对象属性不存在，类型为 undefined。

### 类型判断

- typeof：适用于基本数据类型，但对于 null 和数组等对象类型会返回 "object"。
- instanceof：适用于对象类型，判断一个对象是否是某个构造函数的实例，但对于基本数据类型无效。
- Object.prototype.toString.call()：适用于所有数据类型，返回一个表示对象类型的字符串，如 "[object Array]"、"[object Object]"。
- Array.isArray()：专门用于判断数组类型，返回 true 或 false。

### 变量声明: var, let 与 const

- var：函数作用域，变量提升，允许重复声明。
- let：块作用域，不允许重复声明，存在暂时性死区。
- const：块作用域，不允许重复声明，必须初始化，值不可变（对于对象类型，引用不可变，但对象内容可变），存在暂时性死区。
- 暂时性死区：在块作用域内，使用 let 或 const 声明的变量在声明之前是不可访问的，访问会抛出 ReferenceError 错误。

### 函数

- 函数声明：function foo() {}，存在函数提升，可以在声明之前调用。
- 函数表达式：const foo = function() {}，不存在函数提升，必须在声明之后调用。
- 箭头函数：const foo = () => {}，没有自己的 this、arguments、super 和 new.target，适合简化函数表达式，但不适合用作构造函数。
- 函数默认返回 undefined，如果没有显式返回值。

### 数组方法

- 会改变原数组的方法：push、pop、shift、unshift、splice、sort、reverse。
- 不会改变原数组的方法：concat、slice、map、filter、reduce、forEach、find、findIndex、includes、indexOf、join、toString。

| 遍历方式                                         | 支持异步（await）                     | 支持中断（break/return）                   | 备注                                   |
| -------------------------------------------- | ------------------------------- | ------------------------------------ | ------------------------------------ |
| **for 循环** (`for(let i=0;i<arr.length;i++)`) | ✅ 支持，配合 `await`                 | ✅ 支持 `break` / `continue`            | 最原始、最灵活的方式                           |
| **for...of**                                 | ❌ 不支持异步       | ✅ 支持 `break` / `continue` / `return` | 可直接迭代数组、Set、Map 等可迭代对象               |
| **forEach**                                  | ❌ 不支持异步 `await`                 | ❌ 不支持 `break` / `return`             | 异步需要用 `Promise.all` 或者 `for...of` 替代 |
| **map**                                      | ❌ 不支持异步 `await`（会立即返回新数组）       | ❌ 不支持 `break`                        | 返回新数组，常用于转换；异步要配合 `Promise.all`      |
| **filter**                                   | ❌ 不支持异步 `await`                 | ❌ 不支持 `break`                        | 用于筛选元素，异步可配合 `Promise.all`           |
| **reduce**                                   | ❌ 不直接支持异步 `await`（需手动链 Promise） | ❌ 不支持 `break`                        | 用于累加、聚合；异步需写成 async reduce           |
| **some**                                     | ❌ 不支持异步 `await`                 | ✅ 支持中断（返回 true 停止遍历）                 | 遍历到满足条件即可停止                          |
| **every**                                    | ❌ 不支持异步 `await`                 | ✅ 支持中断（返回 false 停止遍历）                | 遍历到不满足条件即可停止                         |
| **for await...of**                           | ✅ 支持（用于异步迭代器）                   | ✅ 支持 `break` / `continue` / `return` | 用于异步数组或异步生成器（async generator）        |


### DOM 操作与事件模型

- 高效的DOM节点操作：使用 DocumentFragment 批量操作 DOM，减少重排和重绘。
  ```js
  // 假设有一个 ul
  const ul = document.querySelector('ul');
  // 创建 fragment
  const fragment = document.createDocumentFragment();
  // 批量创建 li
  for (let i = 1; i <= 5; i++) {
    const li = document.createElement('li');
    li.textContent = 'Item ' + i;
    fragment.appendChild(li); // 先放在 fragment 中
  }
  // 一次性插入页面
  ul.appendChild(fragment);
  ```
- 事件委托：将事件监听器添加到父元素上，通过事件冒泡机制处理子元素的事件，减少事件监听器数量，提高性能。
- 事件捕获与冒泡：事件传播的两个阶段，捕获阶段从根元素到目标元素，冒泡阶段从目标元素回到根元素。可以通过 addEventListener 的第三个参数指定事件监听器是在捕获阶段还是冒泡阶段触发。
- 捕获阶段：事件从根元素向目标元素传播，先触发父元素的事件监听器，再触发子元素的事件监听器。
- 目标阶段：事件触发在目标元素上，触发目标元素的事件监听器。
- 冒泡阶段：事件从目标元素向根元素传播，先触发子元素的事件监听器，再触发父元素的事件监听器。

### 本地存储方案对比

- localStorage：同步 API，存储容量较大（约 5MB），适合存储较大数据，但可能导致性能问题。
- sessionStorage：同步 API，存储容量较大（约 5MB），数据在页面会话结束时被清除，适合存储临时数据。
- IndexedDB：异步 API，存储容量较大（取决于浏览器），适合存储大量结构化数据，支持事务和索引，但使用较复杂。
- Cookie：同步 API，存储容量较小（约 4KB），数据随每个 HTTP 请求发送，适合存储少量数据，如用户认证信息，但可能存在安全和性能问题。

### script、script async 和 script defer 的区别

- script：默认行为，浏览器会立即下载并执行脚本，阻塞页面渲染，适合需要立即执行的脚本。
- script async：浏览器会异步下载脚本，下载完成后立即执行，可能会在页面渲染完成之前执行，适合独立的脚本，如分析工具。
- script defer：浏览器会异步下载脚本，但会等到页面渲染完成后再执行，适合依赖 DOM 的脚本，如主应用逻辑。

**注意：没有 src 属性的脚本，async 和 defer 属性会被忽略。**

### this 指向

- 全局上下文：在全局作用域中，this 指向全局对象（浏览器中是 window）。
- 函数上下文：在普通函数中，this 的值取决于函数的调用方式。默认情况下，非严格模式下 this 指向全局对象，严格模式下 this 为 undefined。
  - 对象方法：当函数作为对象的方法调用时，this 指向调用该方法的对象。
  - 构造函数：当函数作为构造函数使用（使用 new 关键字调用）时，this 指向新创建的对象。
- 箭头函数：箭头函数没有自己的 this，this 的值由外层作用域决定，通常指向定义时所在的上下文。

### call()、apply()、bind()区别

- call()：立即调用函数，并且可以传入参数列表。
- apply()：立即调用函数，但参数以数组形式传入。
- bind()：返回一个新的函数，绑定 this 和参数，但不立即调用。

### 原型与原型链

- 原型：每个 JavaScript 对象都有一个内部属性 [[Prototype]]，指向另一个对象，这个对象被称为原型。通过原型可以实现属性和方法的共享。
- 原型链：当访问一个对象的属性或方法时，如果该对象没有这个属性或方法，JavaScript 引擎会沿着原型链向上查找，直到找到该属性或方法或者到达原型链的末尾（null）。原型链是 JavaScript 实现继承的一种机制。
- 继承：通过原型链实现对象之间的继承关系，子对象可以访问父对象的属性和方法。
- 准则
  - Person.prototype.constructor == Person // **准则1：原型对象（即Person.prototype）的constructor指向构造函数本身**
  - person01.**proto** == Person.prototype // **准则2：实例（即person01）的**proto**和原型对象指向同一个地方**
- **总结**
  - prototype 是“生产共享方法的地方”
  - [[Prototype]] 是“对象真实的原型指针（内部查找链路）”
  - __proto__ 是“访问 [[Prototype]] 的接口（查找路线）”
  - constructor 是“原型对象指回构造函数的引用指针”

### 继承

- 原型链继承：通过将子类的原型指向父类的实例来实现继承，但存在引用类型属性共享和无法传递参数的问题。
  ```js
    function Parent() {
      this.name = 'Parent';
      this.hobbies = ['coding'];
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {}
    Child.prototype = new Parent();
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    child1.hobbies.push('music');
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent
    console.log(child2.hobbies); // ['coding', 'music']（引用类型属性共享问题）
    child1.sayHello(); // Hello from Parent
    child2.sayHello(); // Hello from Parent

    // child1.__proto__ -> Child.prototype -> Child.prototype.__proto__ -> Parent.prototype

  ```
 
- 构造函数继承：在子类构造函数中调用父类构造函数来实现继承，解决了引用类型属性共享的问题，但无法继承父类原型上的方法。
  ```js
    function Parent() {
      this.name = 'Parent';
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {
      Parent.call(this); // 调用父类构造函数
    }
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent（不共享引用类型属性）
    child1.sayHello(); // TypeError: child1.sayHello is not a function（无法继承父类原型上的方法）
  ```
- 组合继承：结合原型链继承和构造函数继承的优点，既解决了引用类型属性共享的问题，又能继承父类原型上的方法，但存在调用父类构造函数两次的问题。
  ```js
    function Parent() {
      this.name = 'Parent';
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {
      Parent.call(this); // 调用父类构造函数
    }
    Child.prototype = new Parent(); // 创建一个父类实例作为子类原型，调用父类构造函数一次
    Child.prototype.constructor = Child; // 修正 constructor 指向
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent（不共享引用类型属性）
    child1.sayHello(); // Hello from Parent
    child2.sayHello(); // Hello from Parent
  ```
- 寄生组合继承：通过创建一个中间函数来避免调用父类构造函数两次，实现了更高效的继承方式。
  ```js
    function Parent() {
      this.name = 'Parent';
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {
      Parent.call(this); // 调用父类构造函数
    }
    Child.prototype = Object.create(Parent.prototype); // 创建一个中间对象，避免调用父类构造函数两次
    Child.prototype.constructor = Child; // 修正 constructor 指向
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent（不共享引用类型属性）
    child1.sayHello(); // Hello from Parent
    child2.sayHello(); // Hello from Parent
  ```

### 闭包

- 定义：闭包是指一个函数能够访问并操作其外部作用域中的变量，即使外部函数已经执行完毕。
- 作用：闭包可以让函数访问并操作函数外部的变量，即使外部函数已经执行完毕。闭包常用于数据封装、模拟私有变量、实现函数工厂等场景。
- 注意事项：闭包可能导致内存泄漏，因为闭包会保持对外部变量的引用，导致这些变量无法被垃圾回收器回收。因此，在使用闭包时要注意避免不必要的引用，及时释放资源。

### 内存泄漏和内存溢出

| 特性     | 内存泄漏                 | 内存溢出                |
| ------ | -------------------- | ------------------- |
| 定义     | 已不再使用的内存 **未被回收**    | 程序分配内存 **超过环境限制**   |
| 是否立即报错 | 不一定，可能慢慢积累           | 会立即报错或崩溃            |
| 典型原因   | 全局变量、闭包、未移除事件、DOM 引用 | 无限递归、超大对象分配、长时间泄漏累积 |
| 表现     | 内存持续增加、性能下降          | 程序崩溃、报错、堆空间耗尽       |


### Promise

- 定义：Promise 是一种用于处理异步操作的对象，代表一个可能在未来完成或失败的操作及其结果。
- 状态：Promise 有三种状态：pending（等待中）、fulfilled（已完成）和 rejected（已拒绝）。状态只能从 pending 转变为 fulfilled 或 rejected，且一旦状态改变就不能再改变。
- 方法：
  - Promise.all(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的所有 Promise 都 fulfilled 时，返回一个包含所有结果的数组；如果有任何一个 Promise rejected，则返回一个 rejected 的 Promise。
  - Promise.race(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的任意一个 Promise 状态改变时，返回该 Promise 的状态和结果。
  - Promise.allSettled(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的所有 Promise 都 settled（已完成或已拒绝）时，返回一个包含所有结果的数组。
  - Promise.any(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的任意一个 Promise fulfilled 时，返回该 Promise 的结果；如果所有 Promise 都 rejected，则返回一个 rejected 的 Promise。

### async/await

- 定义：async/await 是基于 **Promise 的语法糖**，用于处理异步操作，使异步代码看起来像同步代码。
- async 函数：声明一个函数为异步函数，返回一个 Promise 对象。
- await 表达式：用于等待一个 Promise 对象的结果，必须在 async 函数内部使用。
- 错误处理：可以使用 try/catch 块来捕获 async 函数中的错误，或者在调用 async 函数时使用 .catch() 方法来处理错误。
- **await 不会阻塞主线程，只是暂停 async 函数内部的执行，async 函数永远返回 Promise**

### 高阶函数

- 定义：高阶函数是指接受一个或多个函数作为参数，或者返回一个函数作为结果的函数。
- 作用：高阶函数可以用于函数组合、函数柯里化、函数节流等场景，增强函数的功能和灵活性。
- 示例：Array.prototype.map()、Array.prototype.filter()、Array.prototype.reduce() 等都是高阶函数，因为它们接受一个函数作为参数来处理数组中的每个元素。

### es6 新特性

- let 和 const 变量声明
- 箭头函数
- 模板字符串
- 解构赋值
- 扩展运算符（...）
- 默认参数
- Promise
- 模块化（import/export）
- 类和继承
- Symbol 和 Set/Map 数据结构
- 生成器函数（Generator）
- 对象和数组的扩展（如 Object.assign()、Array.from() 等）
- 数组方法（如 find()、findIndex()、includes()、map()、filter()、reduce() 等）
- Proxy 和 Reflect 对象
- 迭代器和 for...of 循环

### 防抖和节流

- 防抖：一段时间内只执行最后一次（搜索）
- 节流：每隔固定时间执行一次（滚动、窗口调整大小等）

```js
function debounce(fn, delay = 300) {
  let timer = null;

  return function (...args) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
      timer = null;
    }, delay);
  };
}

function debounce(fn, delay = 300, immediate = false) {
  let timer = null;

  return function (...args) {
    if (timer) clearTimeout(timer);
    if (immediate) {
      if (!timer) {
        fn.apply(this, args);
      }
      timer = setTimeout(() => {
        timer = null;
      }, delay);
    } else {
      timer = setTimeout(() => {
        fn.apply(this, args);
        timer = null;
      }, delay);
    }
  };
}

function throttle(fn, delay = 300) {
  let lastTime = 0;

  return function (...args) {
    now = new Date();
    if (now - lastTime >= delay) {
      fn.apply(this, args);
      lastTime = now;
    }
  };
}
```

| 时间(ms) | 事件触发 | 防抖+立即执行                             | 节流                    |
| -------- | -------- | ----------------------------------------- | ----------------------- |
| 0        | 点击一次 | ✅立即执行                                | ✅立即执行              |
| 500      | 点击一次 | ❌不执行（还在等待 delay）                | ❌不执行（间隔未到）    |
| 1200     | 点击一次 | ❌不执行（前一次触发后 delay 已过才重置） | ✅执行（间隔到 1000ms） |
| 1600     | 点击一次 | ✅执行（因为上一次防抖执行 reset timer）  | ❌不执行（间隔未到）    |

### 浅拷贝和深拷贝

- 浅拷贝：只复制对象的第一层属性，如果属性值是引用类型，则复制的是引用地址，修改其中一个对象会影响另一个对象。
  - 常见的浅拷贝方法有 Object.assign()、扩展运算符（...）等。
- 深拷贝：复制对象的所有层级属性，如果属性值是引用类型，则会创建一个新的对象，修改其中一个对象不会影响另一个对象。
  - 常见的深拷贝方法有 JSON.parse(JSON.stringify())、递归函数、第三方库（如 lodash 的 \_.cloneDeep()）等。

### 跨域
**什么是跨域**
- 定义：跨域是指在浏览器中，当前网页的域（协议、域名、端口）与请求资源的域不一致时，浏览器出于安全考虑阻止访问该资源的行为。

**解决方案**

- JSONP：通过动态创建 script 标签来实现跨域请求，适用于 GET 请求，但不支持 POST 请求。
- CORS：通过服务器设置 Access-Control-Allow-Origin 头来允许跨域请求，支持多种 HTTP 方法，但需要服务器端配合。
- 代理服务器：在同域下设置一个代理服务器，将跨域请求转发到目标服务器，适用于开发环境，但需要额外的服务器配置。
- WebSocket：通过建立持久化连接来实现跨域通信，适用于实时数据传输，但需要服务器端支持。

### 垃圾回收机制

- 什么是垃圾回收：垃圾回收是指自动释放不再使用的内存资源的过程，JavaScript 引擎通过垃圾回收机制来管理内存，避免内存泄漏和内存溢出。
- 标记清除算法：JavaScript 引擎使用标记清除算法来进行垃圾回收，该算法通过标记所有可达的对象，然后清除未被标记的对象来释放内存。
- 内存泄漏：内存泄漏是指程序中不再使用的对象仍然被引用，导致无法被垃圾回收器回收，最终可能导致内存溢出。常见的内存泄漏场景包括全局变量、闭包、定时器和事件监听器等。

### 事件循环

- 定义：事件循环是 JavaScript 运行时的一种机制，用于处理异步操作和事件。它通过一个循环不断检查消息队列（Message Queue）和调用栈（Call Stack），确保异步任务在适当的时机执行。
  - 消息队列（浏览器/Node.js）：存放异步任务。
  - 调用栈（js）：存放同步任务，单线程。
- 执行顺序：同步任务->微任务（如 Promise.then()）->宏任务（如 setTimeout()）。当调用栈为空时，事件循环会先执行所有微任务，然后再执行一个宏任务。

### 设计模式

- 单例模式：确保一个类只有一个实例，并提供一个全局访问点。
- 工厂模式：定义一个用于创建对象的接口，让子类决定实例化哪个类。
- 观察者模式：定义对象之间的一对多依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知并自动更新。
- 模块模式：将相关的功能封装在一个模块中，提供公共接口，隐藏内部实现细节。

### JavaScript V8引擎

 - V8 是 Google 开发的一个开源 JavaScript 引擎，用来将 JS 代码编译、执行，支持浏览器（Chrome）和 Node.js。
 - 特点：
   - 高性能：使用 JIT 编译，将 JS 直接编译为本地机器码
   - 垃圾回收：自动回收不再使用的内存
   - 内嵌优化：通过内联缓存（inline caching）、优化编译器等技术提高执行速度
   - 跨平台：不依赖浏览器，Node.js 也能使用