---
title: Vue组件开发实践 - vue-doc-preview
date: 2018-08-14 16:57:47
tags:
---

在写这篇文章的时候，我想起了一句名言，实践是检验真理的唯一标准。我要开始讲故事了，先坐时光机回到上周，上周四的我萌生了写一个Vue组件的想法，正好看了几篇相关的文章，于是我就开始动手写了。写什么组件呢，刚好正在做的一个项目要用到在线预览文档的功能，之前只是简单地实现了几种文本格式文档的预览，这次就做一个相对完善的文档预览组件吧。
<!-- more -->
## 基本实现

项目对于这个组件的需求是这样子的：
* 设定文档的格式（也就是Prop:type），刚开始的设计是支持markdown、text、office三种类型的文档，后来又增加了code类型。
* 设定文档的内容（Prop:value），office文档是传url，其他类型文档直接传内容。

组件的设计并不复杂，我通过拆分出下列几个子组件来处理对应类型的文档：
* Markdown: 处理markdown以及code类型文档的预览，通过[Marked](https://github.com/markedjs/marked)将markdown文本转化为Html，并通过[highlight.js](https://github.com/highlightjs/highlight.js)实现代码高亮。
* Office：通过提供office文档的url给微软的Office Online实现在线预览。
* Text：通过Pre标签实现普通文本的展示。

原来code类的文档（如html、js等后缀的文档）是通过Text组件进行预览处理的，后来为了更好的展示该类文档，将文档内容包装为markdown格式并使用Markdown组件进行处理（通过在文档头部和末尾添加markdown的代码块标识，即可将普通文档包装成markdown的代码块）。同时为了让代码类文档实现代码高亮，又增加了一个Prop:language，用以表明代码的语言。

Office组件的实现使用了iframe，在这里我遇到了一个坑：iframe的高度设置。当iframe的height设置为百分比值时，如果iframe的父容器或（父容器的父容器...）没有设置绝对的高度（如height: 1000px）时，iframe的高度设置是无效的。

为了解决这个问题，我增加了一个新的Prop:height，用于设置组件的高度。为了同时兼容绝对高度和相对高度的设置，在height大于100时组件将会以绝对高度的方式进行设置，否则将以相对高度（百分比）的方式进行设置。不过在Office组件中我采取了一个特殊的处理，在进行相对高度的设置时，组件的高度是相对于当前窗口的高度进行设置的，以避免上面提到的iframe高度设置无效的问题。

这时候组件就基本完成了，在编译（构建）完成，并publish到npm上后，便可以通过yarn/npm等包管理工具引入这个组件了。

## 构建
这里不得不提一下这个组件的构建过程，这个组件我是基于Vue-cli 3.0开始搭建的，除了[官方文档](https://cli.vuejs.org/zh/guide/build-targets.html#%E5%BA%93)上面有一些简单的说明外，网上并没有太多使用cli3.0进行Lib构建的经验分享。幸运的是，经过一番摸索之后，我顺利地完成了这个组件的构建。下面是构建的一些配置（可能有遗漏，完整配置还是需要参照源码）：
```javascript
// package.json中的script
"build-lib": "vue-cli-service build --target lib --name vueDocPreview ./src/index.js"

// package.json中的main指向构建好的文件，这样就可以通过import xxx from 'vue-doc-preview'直接引入组件了
"main": "./dist/vueDocPreview.umd.min.js",

// index.js
import VueDocPreview from './App'
export default VueDocPreview

// vue.config.js
module.exports = {
  css: {
    extract: false // 强制内联，只要构建后的umd包就会包含样式
  }
}
```

## 改进
这个组件在开发到0.2.0版本时，已经可以满足我项目上的需求了，但是其他用户使用这个组件还是可能存在下面这些问题：
1. 难以自定义样式
2. 无法获取文档内容或文档url
3. 构建后的包体积过大

### 自定义样式
Markdown和Text两个组件中都分别内置了一套样式（渲染规则），为了让组件的使用者能更方便的调整这两个组件的样式，我在这个组件上增加了两个Prop:mdStyle和textStyle，采用类似mixins的方式来混入自定义样式（合并默认样式和自定义样式，当发生冲突时采用自定义样式）。

### 文档下载解析
在原来的设计和实现中，除了office文档是通过绑定url进行加载，其他类型文档都是通过绑定文档内容来加载。而获取文档内容通常先要获取文件对象（通过上传或下载），再调用FileReader读取文档的内容。考虑到这个组件的应用场景更多的是预览远程服务器上的文档，因此我引入了axios进行文档的下载。为了支持这一特性，组件新增了Prop:url，组件将根据绑定的url自动下载文档并解析（office文档则直接根据url调用微软Office Online的API）。但是要注意的是，Prop:value的优先级仍然比Prop:url要高，如果同时绑定了value和url，那么组件最终渲染的内容是value中的值，而url则会被忽略。

### 缩小构建包体积
在[v0.3.0](https://github.com/cainsyake/vue-doc-preview/releases)及更新的版本中，由于引入了axios，构建后的体积大了不少（压缩后的umd包比v0.2.2的大了20kb）。如果希望使用更轻量的组件，同时又不需要url下载这一特性的用户可以选择[v0.2.2](https://github.com/cainsyake/vue-doc-preview/releases/tag/v0.2.2)使用。

另外[highlight.js引入](https://github.com/cainsyake/vue-doc-preview/blob/master/src/lib/highlight.js)的大量语言包也会增加构建后umd包的体积，为此我在引入highlight.js时采用了按需引入并注册的方式。用户可根据实际情况只引入并注册需要的语言包，在构建时便可有效减少umd包的体积。
```javascript
// src/lib/highlight.js

import hljs from 'highlight.js/lib/highlight' // 引入highlight.js
import 'highlight.js/styles/arduino-light.css' // 引入样式

/**
 * 按需引入语言包
 */
import javascript from 'highlight.js/lib/languages/javascript'

/**
 * 注册语言包
 * 我们可以设置代码语言的缩写名或别名
 */
hljs.registerLanguage('javascript', javascript)
hljs.registerLanguage('js', javascript)
```

## 结束
在文章的末尾我想放几个关于这个组件的链接：
* [Live Demo | 在线示例](http://vdp.cainsyake.com/)
* [Github: vue-doc-preview](https://github.com/cainsyake/vue-doc-preview)
* [Yarn: vue-doc-preview](https://yarnpkg.com/zh-Hans/package/vue-doc-preview)

各位读者如有建议或意见欢迎提出，我们一共探讨。再次感谢各位读者的阅读，谢谢！