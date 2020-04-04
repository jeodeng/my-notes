## 笔记本
  
1. vue中用了v-if最好使用v-else，不要用v-else-if，否则虚拟dom会渲染两个三目运算符，影响性能。  

2. vue-navigation原理：在内存中自己维护一个栈，使用组件包裹view层，根据hash的Key值把虚拟节点vnode缓存起来。[笔记链接](https://github.com/jeodeng/my-notes/blob/master/articles/vue-navigations.md)  

3. node中正则用了负向先行断言，正向先行这种类似的，在某种环境下不支持，运行在服务器上会报错，尽量避免使用，可用[网址](https://regexr.com/)测试正则。  

4. 今天碰到个坑，node项目之前一直是同事用mac开发的，部署在linux环境，在请求header里面加了个字段，值是new Date()这种日期类型，在mac和linux上接口都没问题，换到我windows环境就报错了，说header contain valid character。估计是windows不会自动把日期类型转成时间戳，而mac和linux都会转，所以开发包括上线运行一直都没问题。后来统一换成时间戳了，就没问题了。  

5. 记录一下帮朋友解决的一个问题。框架vue，一个input输入框，每次输入内容会失去焦点，造成输入一个字符就要点击输入框聚焦。这个问题在本地, mac上都无法重现。我定位问题的思路：环境不同会造成的问题，可能是代码的编译执行，不同环境下会有不同的效果。而且输入框会失去焦点，考虑是否是dom的渲染问题，本地开发时用了webpack-dev-server，并且mac上对比windows可能做了一些优化。验证一下，f12聚焦到input这个dom节点上，开始输入内容。确实，线上环境一旦输入内容，input的dom开始重新渲染，看看代码，这个输入框被一个变量条件v-if判断渲染，查看该变量是否会改变，发现使用了watch, 并且改变变量，可能vue底层发现改变进行了重新渲染，造成页面上input框其实是瞬间消失、出现，失去焦点。把watch给删掉了，bug解决。但我不确定分析的原因是否准确，先打个标记，以后看了vue源码再回头看看该问题是否判断正确。[ ??? ]  

6. 失业了...发现也没累积什么...哎，准备点面试题，不想记在这里，[另开一文](https://github.com/jeodeng/my-notes/blob/master/articles/interview-questions.md)   