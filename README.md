笔记本
======

记录日常笔记

---

#### list

1.	vue中用了v-if最好使用v-else，不要用v-else-if，否则虚拟dom会渲染两个三目运算符，影响性能。
2.	vue-navigation原理：在内存中自己维护一个栈，使用组件包裹view层，根据hash的Key值把dom缓存起来。
