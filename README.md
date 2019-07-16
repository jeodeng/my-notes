笔记本
======

记录日常笔记

---

#### list

1.	vue中用了v-if最好使用v-else，不要用v-else-if，否则虚拟dom会渲染两个三目运算符，影响性能。
2.	vue-navigation原理：在内存中自己维护一个栈，使用组件包裹view层，根据hash的Key值把虚拟节点vnode缓存起来。[笔记链接](https://github.com/jeodeng/my-notes/blob/master/articles/vue-navigations.md)
3.	node中正则用了负向先行断言，正向先行这种类似的，在某种环境下不支持，运行在服务器上会报错，尽量避免使用，可用[网址](https://regexr.com/)测试正则。
