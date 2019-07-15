vue-navigation源码解析
----------------------

[vue-navigation源码链接](https://github.com/zack24q/vue-navigation)

这是一个vue导航插件，可以记录路由并且缓存页面，具体用法请自行了解。比较核心的功能是可以缓存上一个页面，比如填写的一些表单内容。

直接说实现原理：自己维护一个栈用来记录路由的信息，使用vnode特性抽象组件，把所有的组件实例缓存起来。执行back的时候把缓存的实例取出来渲染。

ok，接下来我们看源码。

目录结构:`
    |-- src
        |-- index.js   // 输出的文件
        |-- navigator.js // 封装的组件方法
        |-- routes.js // 路由信息缓存
        |-- utils.js // 工具函数
        |-- components
            |-- Navigation.js // 组件实例
`

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

  // 创建路由
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
