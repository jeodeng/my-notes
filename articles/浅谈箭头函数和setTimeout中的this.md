## 浅谈箭头函数和setTimeout中的this

> 搬运以前写的文章

箭头函数会改变this的指向，这个大家看文档都看到过，可是有没有具体理解呢？  
我自己好像不大理解......emmmm，然后我整理了一遍，加强一下概念吧。  
顺带再讲一下setTimeout这个函数改写this的概念。

首先分别讲一下两位主角

+ 箭头函数：都2020年了，大家肯定不陌生了，用法很简单，可以自行百度，箭头函数有一个很大的特性是会改写内部的this指向，那么实际运用的过程中你考虑过注意过这个问题吗？**箭头函数内部的this会指向声明箭头函数时所在作用域的this**

+ setTimeout：大家肯定都用过了，它的第一个参数是一个方法，**传入的这个方法内部的this会被改写指向window**

一贯风格，我们上代码来看问题
```js
  // 先给window加一个id，以便于确认之后this的指向
  window.id = 0;
  // 声明一个函数fn
  const fn = {
    id: 1,
    say: function() {
      console.log('id:', this.id);
    },
    sayArrow: () => {
      console.log('id:', this.id);
    },
    say1: function() {
      setTimeout(function() {
        console.log('id:', this.id);
      }, 1000);
    },
    say2: function() {
      let that = this;
      setTimeout(function() {
        console.log('id:', that.id);
      }, 1000);
    },
    say3: function() {
      setTimeout(() => {
        console.log('id:', this.id);
      }, 1000);
    },
    say4: () => {
      setTimeout(() => {
        console.log('id:', this.id);
      }, 1000);
    },
    say5: () => {
      setTimeout(function() {
        console.log('id:', this.id);
      }, 1000);
    },
  };
```

接下来做题，不要做晕了

```js
  fn.say();
  fn.sayArrow();
  setTimeout(fn.say, 1000);
  setTimeout(fn.sayArrow, 1000);
  setTimeout(() => fn.say(), 1000);
  setTimeout(() => fn.sayArrow(), 1000);
  fn.say1();
  fn.say2();
  fn.say3();
  fn.say4();
  fn.say5();
```

以上各自输出什么呢？不考虑执行顺序接下来核对下答案，如果全对，那ojbk了  
1 0 0 0 1 0 0 1 1 0 0  
如果觉得自己可能没摸透，可以多包几层作用域再试试  

接下来看代码讲原因！
```
  fn.say();
  /*
    结果: 1
    原因: 通过fn调用的say, say是 函数声明, this指向fn，输出的是fn.id
  */

  fn.sayArrow();
  /*
    结果: 0
    原因: 通过fn调用的say, say是 箭头函数声明, this指向箭头函数声明时作用域的this，也就是this指向window，输出的是window.id
  */

  setTimeout(fn.say, 1000);
  /*
    结果: 0
    原因: 通过setTimeout调用, setTimeout改写所传函数的this, 也就是this指向window，输出的是window.id
  */

  setTimeout(fn.sayArrow, 1000);
  /*
    结果: 0
    原因: 通过setTimeout调用, setTimeout改写所传函数的this, 也就是this指向window，输出的是window.id
  */

  setTimeout(() => fn.say(), 1000);
  /*
    结果: 1
    原因: 通过setTimeout调用, 但是fn.say()是被箭头函数包裹，所以fn.say()调用不受this改变的影响，原因参考第一句输出
    ps: 等同于 setTimeout(function () { fn.say(); } , 1000);
  */

  setTimeout(() => fn.sayArrow(), 1000);
  /*
    结果: 0
    原因: 同上雷同，具体原因可参考第二句输出
    ps: 等同于 setTimeout(function () { fn.sayArrow(); } , 1000);
  */

  fn.say1();
  /*
    结果: 0
    原因: setTimeout改写函数内部的this, 使其指向window, 输出window.id
  */

  fn.say2(); // 1
  /*
    结果: 1
    原因: setTimeout虽然改写函数内部的this,但是输出的是that.id，这个that在setTimout外面声明，指向的是fn，所以输出fn.id
  */

  fn.say3(); // 1
  /*
    结果: 1
    原因: fn.say3函数其内部的this指向的fn，而setTimeout内部传了一个箭头函数，箭头函数内部中的this就指向了fn.say3内的this，也就是fn，最后输出fn.id
  */

  fn.say4(); // 0
  /*
    结果: 0
    原因: fn.say4这个函数本身就是使用箭头函数声明，其内部的this指向的是箭头函数声明时所在的作用域，即window，而且setTimeout也是传了一个箭头函数，这里面的this指向外层箭头函数内部中的this，传引用也指向了window，最后输出window.id
  */

  fn.say5(); // 0
  /*
    结果: 0
    原因: fn.say5用箭头函数声明，其内部的this指向window，而且setTimeout也会改写内部this指向window，最后输出window.id
  */
```

其实有一些调用其根本原因是相同的，但是我个人喜欢各个方面去验证一下，加深一下印象也好，勿喷。  
附带说一嘴，如果有同学在学node, node内部是没有window对象的，window是浏览器规范，所以具体情况看你在哪里用了，不过概念是差不多的  

写完觉得这篇文章真的是浅谈...好low啊哈哈哈哈，写给自己看！