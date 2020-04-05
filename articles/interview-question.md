# 面试题记录

我会尽量写得详细一些，老少皆宜

## JS系列

### 1. call, apply, bind区别，手撕bind实现

这三者的目的都是一致的：为了改变this的指向。比如声明一个函数fn，

```js
  function fn(sex, age) {
    console.log(`我叫${this.name}, 性别${sex}, 今年${age}岁`);
    return 'yes！';
  }

  window.name = 'window';

  fn('人妖', 18); // 我叫window，性别人妖，今年18岁
```
如果直接调用fn, 那么输出的是 window，因为this指向的是window。如果想让fn可以随便输出指定对象的name？直接代码看分别如何使用

```js
  const aa = { name: '张三' };
  const bb = { name: '李四' };
  const cc = { name: '王五' };

  fn.call(aa, '男', 19); // 我叫张三，性别男，今年19岁
  fn.apply(bb, ['女', 20]); // 我叫李四，性别女，今年20岁

  fn.bind(cc)('男', 21); // 我叫王五，性别男，今年21岁
  fn.bind(cc, '男', 21)(); // 我叫王五，性别男，今年21岁
  fn.bind(cc, '男')(21); // 我叫王五，性别男，今年21岁
```
区别显而易见，call和apply都会直接调用fn，bind是会返回一个改变函数内部this指向的函数，需要再次调用。  并且这三者在传参上有所不同，第一个参数均为改变的this对象，call是参数列表，apply是参数数组，bind可以先传参数，也可以在函数调用的时候在传。根据这些特性，实际开发中使用也都有不同情景。  

如何手动实现bind呢？上代码
```js
  // 在Function原型链上扩展，让函数都具备该方法
  // ctx也就是this要指向的对象

  Function.prototype.bindHandler = function (ctx) {
    // self其实就是函数
    const self = this;

    return function() {
      self.apply(ctx);
    }
  }

  // 试验一下
  const obj = { name: 'jeo' };
  fn.bindHandler(obj)('男', 24); // jeo, undefined, undefined
  
  // this是改了，参数没有拿到
  // 而且还有一个点没考虑到，fn函数如果返回一个值，能拿到吗？明显不能，改进一下函数
  
  Function.prototype.bindHandler = function (ctx) {
    const self = this;

    // arguments为函数调用时传进来的所有参数集合
    // 保存一下除了ctx以外传进来的参数
    // 因为arguments是伪数组，所以直接用Array原型链上的方法调用
    const args = Array.prototype.slice.apply(arguments, [1]);
    
    return function() {
        
      // 这个arguments和上面的arguments不一样，指的是返回的这个函数拿到的arguments
      const argsOther = Array.prototype.slice.apply(arguments);

      // 把两个拿到的arguments给组合起来，并且拿到函数的返回值
      const result = self.apply(ctx, args.concat(argsOther));
      return result;
    }
  }

  // 再试一下, 完全objk。
  fn.bindHandler(obj, '男', 3)();
  fn.bindHandler(obj)('男', 18);
```

### 2. 什么是闭包，手写一个闭包

闭包这个问题呢牵涉到作用域的知识，我对于闭包的理解就是：**一个函数记住并且能够访问他所在的此法作用域，在其他的作用域调用该函数，就会产生闭包**。用一段简单代码简单说明下这句话。

```js
  function fn () {
    const str = 'HelloWorld';
    function bar () {
      console.log(str);
    };
    return bar;
  }

  var fo = fn();
  fo() // 输出HelloWorld;
```

fo其实就是fn作用域中的函数bar，但是fo在调用的时候并不是在fn内部作用域中调用，即使如此依旧访问到了str这个变量，在这里产生了闭包。  

那么闭包有啥用呢？ 仔细想想自己的代码，闭包无处不在，比较明显运用闭包的例子就是用闭包实现"**数据缓存**"了，看下一题。

### 3. 使用闭包实现数据缓存

```js
  const foo = (() => {
    const cache = {};
    return {
      getCache(key) {
        return cache[key];
      },
      setCache(key, val) {
        cache[key] = val;
      },
      delCache(key) {
        delete cache[key];
      },
    };
  })();

  foo.setCache('name', 'jeo')；
  console.log(foo.getCache('name')); // jeo
  foo.delCache('name');
  console.log(foo.getCache('name')); // undefined
```
这样很简单的就实现了一个数据缓存，也不会污染外部变量。  
那么在哪里体现了闭包呢？  
直接用IIEF返回了一个对象，只暴露方法，在这里就用了闭包，函数调用时所在的域并不是函数声明所在的域，依旧访问到了想要的变量，产生了闭包。

### 4. JS的数据类型，基本类型哪些，和引用类型的区别

数据类型： null, undefined, Number, String, Object, Boolean, Symbol
基本类型： undefined, null, boolean, number, String, 指的是保存在栈内存中的简单数据
引用类型： Array, Object等对象类型，指的是保存在堆内存中的对象，变量实际保存的是一个指针，指针指向内存中的一个位置，该位置保存对象。  

ES6中新引入的Symbol也是基本类型

### 5. 防抖和节流，有什么区别，谈谈应用场景和如何实现？
我们日常开发过程中，经常会碰到频繁触发某事件的场景。举个例子：页面有一个查询按钮，如果用户1秒内疯狂点击该按钮会发送n个查询请求给服务器，这样容易给服务器造成不必要的性能压力。其实只要控制在这1秒内只发送一次查询请求给服务器即可。

**而防抖和节流的作用都是防止函数多次调用的**。

区别在于： 在一个时间间隔 *T* 内触发多次函数，防抖只会调用一次函数，节流则是将多次调用变成每隔一段时间 *T* 触发函数。

如何实现呢？

#### 防抖 

一般有两种执行方式  
延迟执行： T 毫秒内触发多次函数，均会在最后一次触发的 T 毫秒后执行  
立即执行： 触发函数的时候立即执行，'连续触发函数的时间段 + T毫秒' 内均不会再次执行该函数

```js
/**
 * debounce 防抖函数
 *
 * @param  {function}   func        要防抖的函数
 * @param  {number}     delay       毫秒， 指上文时间间隔T
 * @param  {Boolean}    immediate   是否立即调用
 */

function debounce(fuc, delay = 300, immediate = true) {
  let timer, self, args;

  // 创建延迟执行，会返回一个延迟执行函数的定时器
  const newTimer = () => setTimeout(() => {
    
    // 清空定时器，方便下次重新计时
    timer = null;

    // 如果是延迟执行，直接执行函数
    if (immediate === false) {
      fuc.apply(self, args);
      
      // 清空之前闭包缓存的this和参数
      self = null;
      args = null;
    }
  }, delay);

  return function () {
    // 拿到调用函数的所有参数
    const argsOther = Array.prototype.slice.apply(arguments);

    // 如果没有定时器，说明第一次调用函数，创建一个定时器
    if (!timer) {

      timer = newTimer();

      // 立即执行函数
      if (immediate) {
        fuc.apply(this, argsOther);
      } else {
        // 如果是延迟执行函数，则缓存this和arguments
        self = this;
        args = argsOther;
      }
    } else {
      // 已有定时器，说明是在重复调用函数，则重新开始计时
      clearTimeout(timer);
      timer = newTimer();
    }
  };
}

// 如果要给一个div绑定鼠标滑动事件，直接使用
function move () {
  console.log(new Date())
}
div.onmousemove = debounce(move);
```

#### 节流

夜深了...我困了...写个简单版的...  
原理就是重复触发函数的时候，一段时间内只能执行一次函数，就是加了个锁
```js
function throttle(fuc, delay = 300) {
  let timer = 0;
  return function fn() {
    const self = this;
    const now = +new Date();
    const args = Array.prototype.slice.apply(arguments);
    if (now - timer > delay) {
      fuc.apply(self, args);
      timer = now;
    }
  };
}
```

## Vue系列

### 1. Vue在使用v-for渲染列表组件的时候推荐给组件绑定key，key有什么用？

使用列表数据data渲染组件的时候，data发生变化，vue会采用diff算法去比较新旧虚拟节点，根据比较的结果 复用 > 更新 节点。key的作用就是为了，在diff的过程中，能更快更准确的拿到对应的vnode（虚拟节点）对象，减少比对的时间，在某种使用情景下可以提高效率。为了更好理解，分别说一下两种情况：

**不带key**

+ **直接复用节点**。 因为diff比较的时候，key为undefined，那么直接就判断为相同的，所以直接复用，并不会因为遍历到的对象不同而创建/删除节点，所以在做简单列表组件的渲染上，不带key渲染速度会快一些，因为简单所以不需要去考虑其他。

+ **无法维持组件状态**。实际运用的过程中，经常组件嵌套组件，组件本身还维持着别的组件状态，这种复杂情况的使用下，不带key会导致一些不可预知的错误，比如嵌套子数据无法更新等。 

+ **可能渲染性能降低**。之前说会直接复用节点，如果该组件在开发过程中，嵌套渲染其他子组件，在直接复用节点的前提下，需要创建/修改的子组件就可能会很多，性能就会下降。

**带key**

+ **根据key找到对应节点复用，维持旧组件的状态**。

+ **查找性能上的提升**。带唯一key查找和比较查找的区别...不用多说了。

+ **节点复用，性能可能提升**。复杂组件在重新渲染时候复用的话，子节点也可以直接复用，性能就会提升了。

### 2. bbb

### 3. ccc

## CSS系列

### 1. aaa

### 2. bbb

### 3. ccc

## 其他

### 1. 输入URL -> 页面生成 全过程

从两个层面分析：  

#### 网络层面  
简单概况：进行DNS解析 => 拿到IP => 建立连接 => 传输资源 => 断开连接
1. 浏览器先会在浏览器缓存中去查找该域名是否有对应的IP信息
2. 接着搜索操作系统中有没有相应的缓存信息，比如hosts文件中
3. 接着在路由器缓存中进行查找
4. 接着从网络服务商那边的缓存信息中进行查找
5. 以上方法如果找到就直接返回IP信息，如果都没有找到就会发起一个迭代DNS解析请求：  
    1. 向根域名服务器发送请求，该服务器没有域名的全部信息，但是可以返回顶级域名服务器地址，如com域的顶级域名服务器地址；  
    2. 接着向顶级域名服务器发送请求最终获取IP地址；
6. 本地域名服务器（也就是网络服务商）将IP返回给操作系统同时把该IP缓存起来
7. 操作系统把IP返给浏览器，同时把IP缓存起来
8. 最后浏览器得到了IP开始建立连接
9. 建立连接 => 3次握手  
    1. 客户端发送带有SYN标志的数据包给服务端（请求连接）
    2. 服务端发送带有SYN/ACK标志的数据包给客户端（同意建立连接）  
    3. 客户端发送带有ACK标志的数据包给服务端（开始连接）  
  
    PS：如果三次握手中某次某一方没接到信号，TCP协议会要求重新发送信号
10. 建立连接后会开始数据的传递，比如相应的js文件，css文件，图片文件等。同时浏览器开始渲染页面  
11. 当数据传递全部完成，开始四次挥手，断开连接。

#### 页面渲染层面
1. 解析HTML，遍历文档节点生成DOM树;
2. 解析CSS文件生成 CSSOM规则树;
3. 然后DOM树和和CSSOM树结合生成渲染Render树；
4. 渲染树开始布局，计算每个节点的位置大小等信息；
5. 然后绘制每个节点到屏幕上，呈现页面；  

补充：  
  在生成DOM树的过程中，如果遇到script标签，会产生阻塞；  
  如果JS脚本中还对CSSOM进行了操作，浏览器会延迟脚本的执行和DOM的渲染，先生成CSSOM树；  
  渲染树进行绘制的时候，涉及到了重绘和回流：
   - 回流：当元素的大小布局等发生改变的时候就会触发回流，回流必然引发重绘。
   - 重绘：当元素的颜色背景之类发生改变的时候触发重绘。
  
  每个页面必然发生一次回流，所以必然有一次重绘，也就是页面加载渲染的时候；  
  因为回流会导致渲染树重新构建，花销较大，所以应该尽可能的避免回流，比如减少DOM操作等。  