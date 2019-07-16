vue-navigation源码解析
----------------------

[vue-navigation源码链接](https://github.com/zack24q/vue-navigation)

请大家和源码一起阅读本篇文章。

这是一个vue导航插件，可以记录路由并且缓存页面，具体用法请自行了解。比较核心的功能是可以缓存上一个页面，比如填写的一些表单内容。

直接说实现原理：自己维护一个`栈`用来记录路由的信息，使用vnode特性抽象组件，把所有的组件实例缓存起来。执行back的时候把缓存的实例取出来渲染。

ok，接下来我们看源码。

目录结构:

```
|-- src
    |-- index.js   // 输出的文件
    |-- navigator.js // 封装的组件方法
    |-- routes.js // 路由信息缓存
    |-- utils.js // 工具函数
    |-- components
        |-- Navigation.js // 组件实例
```

首先看`index.js`，顶部引入了一些文件

```js
// 存储路由信息的变量，里面代码很简单，不做过多解释
import Routes from './routes'

 // 引入navigation的一些方法，在路由钩子里面会使用
import Navigator from './navigator'

// 引入navigation组件，用来缓存vnode
import NavComponent from './components/Navigation'

// 工具方法，genKey：生成路由key  isObjEqual: 判断两个对象是否具备相同属性
import { genKey, isObjEqual } from './utils'
```

`routes.js`和`utils.js`都很简单，看一眼就能明白干啥的。不多说，接下来其实就是导出一个对象，该对象具有install属性，是一个方法。这是Vue插件的写法，使用Vue.use(xxx)会去调用install方法。

```js

 /*
    在这解释一下传参
    第一个参数是Vue实例 （必传）
    第二个参数是一个对象，该对象有多个属性
  */

install: (Vue, { router, store, moduleName = 'navigation', keyName = 'VNK' } = {}) => {

  // 判断是否传入路由，若无抛错
  if (!router) {
    console.error('vue-navigation need options: router')
    return
  }
  // 创建承载容器
  const bus = new Vue()

  // 创建导航
  // moduleName为store中的module名，默认navigation
  // keyName 在url中的key名，默认VNK
  const navigator = Navigator(bus, store, moduleName, keyName);

  // 改写router的replace方法
  const routerReplace = router.replace.bind(router);
  let replaceFlag = false
  router.replace = (location, onComplete, onAbort) => {
    replaceFlag = true
    routerReplace(location, onComplete, onAbort)
  }

  // 包装了个路由钩子，初始化key
  router.beforeEach((to, from, next) => {

    // 判断下一级路由是否有key
    if (!to.query[keyName]) {

      const query = { ...to.query }

      // isObjEqual用于判断两个对象是否相同
      // 对下一级路由进行处理
      // 跳到相同路由不改变key
      if (to.path === from.path && isObjEqual(
        { ...to.query, [keyName]: null },
        { ...from.query, [keyName]: null },
      ) && from.query[keyName]) {
        query[keyName] = from.query[keyName]
      } else {
        query[keyName] = genKey();
      }

      // 打断当前导航，带上参数再次进入路由
      next({ name: to.name, params: to.params, query, replace: replaceFlag || !from.query[keyName] })
    } else {
      next();
    }
  })

  // 记录路由的变化
  router.afterEach((to, from) => {
    navigator.record(to, from, replaceFlag)
    replaceFlag = false
  })

  // 注册组件
  Vue.component('navigation', NavComponent(keyName))

  // 写入vue全局方法
  Vue.navigation = Vue.prototype.$navigation = {
    on: (event, callback) => {
      bus.$on(event, callback)
    },
    once: (event, callback) => {
      bus.$once(event, callback)
    },
    off: (event, callback) => {
      bus.$off(event, callback)
    },
    getRoutes: () => Routes.slice(),
    cleanRoutes: () => navigator.reset()
  }
}
```

这个文件都是很基本的插件要做的事，接下来看看具体逻辑的实现`navigation.js`，这个里面的代码很简单，只说几个核心的点，其他的仔细看看就能明白。这个文件输出一个函数，函数会返回一个nav对象，对象里有两个函数。核心是`record`这个函数。该方法会根据操作调用不同的函数。

```js
 // toRoute和fromRoute其实就对应vue-router的守卫钩子里面的to, from

 const record = (toRoute, fromRoute, replaceFlag) => {
   // 根据路由信息获取key值
   const name = getKey(toRoute, keyName);
   // 类型判断
   // 如果是替换，直接replace
   if (replaceFlag) {
     replace(name, toRoute, fromRoute)
   }
   else {
     // 在栈中从后往前找
     const toIndex = Routes.lastIndexOf(name)
     if (toIndex === -1) {
        // 找不到，相当于进入一个新的页面，前进
        forward(name, toRoute, fromRoute)
      } else if (toIndex === Routes.length - 1) {
        // 要去的路由是栈最后一个，代表刷新
        refresh(toRoute, fromRoute)
      } else {
        // 返回
        back(Routes.length - 1 - toIndex, toRoute, fromRoute)
      }
    }
  }
```

每次的路由的钩子里都会调用该方法，来执行不同的方法记录路由信息。简单的说一个`forward`方法：

```js
  const forward = (name, toRoute, fromRoute) => {
    const to = { route: toRoute }
    const from = { route: fromRoute }

    // 判断store是否存在
    const routes = store ? store.state[moduleName].routes : Routes
    // 非空判断
    from.name = routes[routes.length - 1] || null
    to.name = name

    // 三目判断，执行store的方法还是还是直接改变routes
    store ? store.commit('navigation/FORWARD', { to, from, name }) : routes.push(name)

    // 把记录存到sessionStorage里面
    window.sessionStorage.VUE_NAVIGATION = JSON.stringify(routes)
    bus.$emit('forward', to, from)
  }
```

`routes`和`store.state`里面的那个`routes`指向的是同一个地址。每个方法其实都差不多，增删改之类的操作。这些方法的目的只有一个，在内存中维护一个`栈`，用来记录每个路由的历史key。

接下来要说另一个核心的点：缓存页面。使用到了组件的特性，把虚拟的组件实例给缓存了起来，通过key值在路由back的时候再次渲染相应的实例。移步到`components/Navigaton.js`这个文件。

这个组件不需要dom或者css，所以直接一个函数，使用`render`去渲染组件。

`watch`了`routes`的变化就是在保证路由信息和缓存的实例匹配，避免不必要的缓存。

`destroyed`的目的就是在组件销毁的时候也销毁缓存。

```js
render() {
  // 判断该组件有没有内容
  const vnode = this.$slots.default ? this.$slots.default[0] : null
  if (vnode) {

    // 给该节点的key赋值
    vnode.key = vnode.key || (vnode.isComment
      ? 'comment'
      : vnode.tag)

    // 根据路由获取hash key
    const key = getKey(this.$route, keyName)

    // 判断 如果该节点key属性中不存在key，则加上
    if (vnode.key.indexOf(key) === -1) {
      vnode.key = `__navigation-${key}-${vnode.key}`
    }

    // 判断缓存中是否存在该节点
    if (this.cache[key]) {
      if (vnode.key === this.cache[key].key) {
        // 判断要渲染的节点key是否存在缓存中，如果存在就从缓存中取出来恢复该节点
        vnode.componentInstance = this.cache[key].componentInstance
      } else {
        // 替换要缓存的实例
        this.cache[key].componentInstance.$destroy()
        this.cache[key] = vnode
      }
    } else {
      // 缓存一个新节点
      this.cache[key] = vnode
    }
    vnode.data.keepAlive = true
  }

  return vnode
}
```

如果想要深入了解`render`和`vnode`相关方面的东西，可以看看[这篇文章](http://hcysun.me/vue-design/zh/renderer.html#%E8%B4%A3%E4%BB%BB%E9%87%8D%E5%A4%A7%E7%9A%84%E6%B8%B2%E6%9F%93%E5%99%A8)
