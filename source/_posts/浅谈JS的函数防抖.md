---
title: 浅谈JS的函数防抖
date: 2018-07-22 13:09:31
tags:
---
前阵子我在写一个注册组件的时候看到Vue中文社区推了一篇关于js的函数防抖和函数节流的文章。刚好我在写的那个组件要用到表单验证，于是我就开始试着实现了函数防抖。
<!-- more -->
### 函数防抖（debounce）
还是先说一下函数防抖的含义吧，当一个函数被调用后(state1)，它会延迟到n毫秒后执行(state2)，如果在state1-->state2之间这个函数被再次调用，那么上一次的调用将会被取消，并再次被延迟到n毫秒后执行。<br>
举个具体点的例子：<br>
有一个input是注册时接收用户输入的用户名的，我希望在用户输入完用户名后对用户名进行校验（检查该用户名是否被占用），于是我通过onchange事件去监听用户输入的变化并调用校验方法。但是用户在input框中每输入一个字符都会触发一次onchange事件，这样会造成计算资源的浪费，也可能造成页面提示信息快速闪烁。<br>
这时候就可以利用函数防抖进行优化了，当input的onchange事件被触发后，首先通过clearTimeout()方法取消上一次由setTimeout()方法设置的延迟执行函数。然后通过setTimeout方法设置一个延迟2秒后执行的函数。这两步操作可以使得用户在input中连续输入字符时并不会频繁调用校验方法，而是当用户输入完2秒后才调用校验方法。<br><br>
示范代码如下(Vue)
```javascript
<template>
  <div>
    <input type="text" v-model="content" @change="inputChange">
  </div>
</template>

<script>
import MarkdownEditor from '_c/markdown'
export default {
  name: 'markdown_page',
  data () {
    return {
      content: '',
      timer: null   // 定时器标识符
    }
  },
  methods: {
      inputChange: function () {
          let self = this
          clearTimeout(self.timer)  // 取消上一次延迟执行的代码
          self.timer = setTimeout(() => {
              console.log(self.content)
          }, 2000)
      }
  }
}
</script>
```

### 函数节流（throttle）
既然说到函数防抖，那就不得不提一下函数节流了。<br>
函数节流的概念：规定一个单位时间，在这个单位时间内，只能有一次触发事件的回调函数执行，如果在同一个单位时间内某事件被触发多次，只有一次能生效。<br>
例如在监听mousemove事件时，每移动一个像素都会触发一次，频繁的函数将会消耗大量资源，这时候我们就需要通过函数节流来降低一个函数执行的频率。使得在一个设定的时间段内只执行一次函数。<br><br>
关于函数防抖和函数节流的更多细节可以参考下面几篇文章。<br>
* [轻松理解JS函数节流和函数防抖](https://mp.weixin.qq.com/s/3FZJ0nQLhj9PCi0pfBjc9A)
* [JS函数节流和函数防抖问题分析](https://www.jb51.net/article/130840.htm)
* [JavaScript函数节流和函数防抖之间的区别](https://www.cnblogs.com/walls/p/6399837.html)
