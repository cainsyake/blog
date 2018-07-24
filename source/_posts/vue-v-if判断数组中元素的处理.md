---
title: vue v-if判断数组中元素的处理
date: 2017-12-17 16:01:17
tags:
---
我的vue版本为2.8
为了实现评论回复的功能，我用v-for遍历了每一条评论。然后每条评论里面都有一个div放**回复**(reply)的输入框和按钮，并用v-if控制这个div的显示。

大概逻辑如下：
点击**回复按钮** -> this.allowReply[index] = true -> 显示**回复div**
<!-- more -->
但在实践中发现即便将对应的控制变量赋值为true后，对应的**回复div**还是没有显示出来。
经过搜索，发现了差不多的情况，[点击这里查看问题](https://stackoverflow.com/questions/41580617/vuejs-v-if-arrayindex-is-not-working)

看了解决办法后，我使用**splice()函数**后，就生效了，但还是遇到了一些**Bug**。
```javascript
this.allowReply.splice(index, 1, true);  //似乎这样可将allowReply数组中下标为index处的值改为true
```
当index**大于0**时，true会被插入allowReply[allowReply.length]，而不是allowReply[index]上。
（即当**allowReply为空数组**时，this.allowReply.splice(2, 1, true) 只会将true插入到**allowReply[0]**中，而不是**allowReply[2]**里面）

为了解决这个问题，我在获取评论数据后，把allowReply用false填充满，代码如下：
```javascript
for (var i = 0; i < this.commentData.length; i++) {
    this.allowReply.push(false);
}
```
之后再执行splice函数对allowReply中的元素进行操作时，便不会出现异常，但这也增加了性能消耗。
希望能尽快找到更好的解决办法。