## 浅谈async函数await用法

> 搬运以前写的文章

async和await相信大家应该不陌生，让异步处理变得更友好。

其实这玩意儿就是个Generator的语法糖，想深入学习得去看看Generator，不然你可能只停留在会用的阶段。

用法很简单，看代码吧。
```js
  // 先声明一个函数，这个函数返回一个promise, 先记住哈！后面很多地方要用
  function getPromise(str = 'sucess') {
    
    return new Promise((resolve) => {
      setTimeout(() => {
        console.log('执行了一个异步');
        resolve(str)
      }, 1000);
    });
  }

  // async表示，这个函数有异步操作！
  async function fn() {
    // getPromise会返回一个Promise
    const data = await getPromise();
    // fn运行在这停顿，这里会停1秒，最后输出data
    // 要wait等待getPromise()这个异步操作返回结果
    console.log(data, 'data');

    // 最后返回data，当然你要是处理完业务也可以不返回
    // 视场景而定了，只是想告诉你async会返回一个promise，而这个data在then里面拿到
    return data;
  }

  fn().then(res => console.log(res) 'res');

  // 这段代码运行出来两个sucess
```

我觉得async最大的好处就是，代码结构更清晰，有更好的语义，写复杂业务的时候阅读起来更快更爽。

接下来模拟一个实际项目的业务场景来看看用法区别

业务场景：我们有一本书，目前只有书名
要通过请求 getBookId 获取到书的id
然后靠id通过请求 getBookDes 获取到书的description
最后要把id，和title，还有description一起存到数据库中 uploadBookInfo

不要纠结http请求如何封装哈，这里我直接给几个模拟例子让同学们方便试.

```js
  // 获取书籍Id
  function getBookId() {
    return new Promise((resolve) => {
      setTimeout(() => resolve('1001'), 1000);
    });
  }

  // 获取书籍描述
  function getBookDes() {
    return new Promise((resolve) => {
      setTimeout(() => resolve('这是一本好书'), 1000);
    });
  }

  // 上传书籍信息
  function uploadBookInfo() {
    return new Promise((resolve) => {
      setTimeout(() => resolve('上传成功'), 1000);
    });
  }

  // promise写法
  function uploadWidthPromise(title = '你不知道的JavaScript') {
    this.getBookId(title).then((id) => {
      console.log(id); // 1001
      this.getBookDes(id).then((des) => {
        console.log(des); // 这是一本好书
        this.uploadBookInfo({
          title,
          id,
          des,
        }).then((res) => {
          console.log(res); // 上传成功
        });
      });
    });
  }

  // async写法
  async function uploadWidthAsync(title = '你不知道的JavaScript') {
    const id = await this.getBookId(title);
    const des = await this.getBookDes(id);
    const result = await this.uploadBookInfo({ id, des, title });
    console.log(id, des, result); // 1001 这是一本好书 上传成功
  }
```

这明显的差距啊，以前用回调，后来用promise觉得这个then可真好用啊，异步完了我就then里面接着写，多清晰！

现在有了await，真香！

而且用await你会发现你的代码执行下来，看起来就像是由上往下执行的顺序，一眼就看完这些干了啥。

 
接下来要说几点用async函数过程中要注意的东西, 划重点啦！！

### 1. 错误捕捉
await语句后面跟着的promise对象一旦抛出错误，也就是变成reject状态，那么整个async函数就会停止执行抛出错误。什么意思呢？

```js
async function thorwErr() {
  await Promise.reject('出错');
  console.log('执行了吗？'); // 不会执行，以下代码都不会执行
  return await Promise.resolve('成功');
},

thorwErr().then((res) => {
  console.log(res); 
  // 成功，并不会弹出，因为第一句awiat已经抛错，被下面的catch捕获，而且async直接停止执行
}).catch((err) => {
  console.log(err); // 出错
});

// 最后只会输出两个字 出错
```

那么这种情况有时候是不符合业务逻辑的，如果我们希望第一句即使出错也不会中断，那么我们需要用到一个try ... catch，如下
```js
  async function thorwErr() {
    try {
      await Promise.reject('出错');
    } catch(err) {
      console.log(err); // 出错
    }
    console.log('执行了吗？');
    return await Promise.resolve('成功');
  }
```

这样写就会被try...catch捕获错误，而不会被async的catch捕获造成函数停止执行

最后输出的也是 出错 执行了吗 成功 这样的三句话

当然也可以换种方式写，如下
```js
  async function thorwErr() {
    await Promise.reject('出错').catch(err => console.log(err));
    console.log('执行了吗？');
    return await Promise.resolve('成功');
  }
```

这样写也ok，道理是一个道理。错误内部直接处理了，不抛给async函数。

在看 阮一峰的ES6 的时候还看到一个例子，我觉得不错分享给大家。

实现了多次尝试请求。也许会有这种情景，一个第三方接口不太稳定，可能要多次调用才会成功一次，就可以用这种方案解决。

```js
  // 尝试反复做某事
  async function tryDoSomething(limit = 3) {
    const limit_num = parseInt(limit, 10); //反复尝试次数 3次

    let i, result;

    for (i = 0; i < limit_num; i++) {
      try {
        // 如果await成功，就会执行break，失败就中断被catch捕捉，再次进入循环
        result = await getSomething();
        break;
      } catch(err) {
        console.log(err);
      }
    }

    return result;
  }
```

### 2. await只能用于async函数的域里面 ！！

```js
async function fn() {
  let arr = [1, 2, 3];

  // 这里就报错了 await is a reserved word
  arr.forEach((i) => {
    await getPromise(i);
  });
  // 因为await其实是在一个箭头函数里面，并不是用在async函数里面
}
```

那么正确的写法如下，也可以理解为await最近的父级函数必须是async函数

```js
function fn() {
  let arr = [1, 2, 3];

  arr.forEach(async (i) => {
    await this.getPromise(i);
  });
}
```

当然，上面这种写法会有另外一个问题，循环三次执行3次await，但是这三个是并发执行，也就是同一时间执行，而不是继发执行，这里就要说到我们第三个要注意的点，**并发执行和继发执行**（！！划重点），往下面看。

### 3. await的继发执行和并发执行

我们经常会碰到的一种业务场景，一个页面要调3个接口，展示3块数据。那么如果我用await岂不是要一个一个的等？这样非常耗时，那么我们可以这么写。

```js
// 这里的getPromise请看文章最开始的声明
const [res1, res2] = await Promise.all([getPromise(1), getPromise(2)]);
```

以上写法是并发执行，这样我们就同时做两个异步操作并且拿到返回的数据了。

循环中继发执行怎么做呢？正确的做法是用for循环

```js
async function fn() {
  const arr = ['1', '2', '3'];
  console.time('耗时：');
  for(let str of arr) {
    console.log(await this.getPromise(str)); // 每隔1秒输出，继发执行
  }
  console.timeEnd('耗时：'); // 这里可以看到比上次输出 有3秒之差
}
```
为了大家更直观的比较，在这里我再写一个循环的并发执行

```js
  async function fn() {
    const arr = ['1', '2', '3'];
    console.time('耗时：');
    for (let promise of arr.map(str => this.getPromise(str))) {
      console.log(await promise); // 同一时间输出 1，2，3，并发执行
    }
    console.timeEnd('耗时：'); // 这里可以看到比上次输出 仅有1秒
  }
```

个人不推荐在循环中使用await，如果要循环使用也用for of继发执行，并发执行用Promise.all的写法了，也是返回一个Promise也可以用await。养成这种习惯，比较方便区分，阅读代码也一目了然。