---
title: Node.js搭建分布式PDF渲染服务
date: 2019-08-12 00:07:12
tags:
---

前段时间一直在忙这个项目，前前后后做了差不多两个月，现在终于有时间整理一下了。
公司的业务系统需要前端生成各类报表的**PDF文件**。以前是使用**jspdf** + **html2canvas**实现的，但是也存在不少问题：
1. 生成的PDF文件体积过大，如果报表尺寸较大或者需要提高分辨率时，PDF文件的大小会急速增长。
2. 在客户端进行文件生成，存在浏览器兼容问题。
在为了解决上面的这两个问题而烦恼时，突然想起了浏览器自带的打印功能。
<!-- more -->
## 技术选型
使用浏览器打印一个10页A4尺寸的PDF文件，文件大小仅在1M左右，而且PDF文件中的文字能够保持为矢量。

后来经过一番调研，决定采用**Node.js** + **Puppeteer**来为公司的业务系统提供PDF渲染服务。
Node.js就不用多作介绍了，Puppeteer是Chrome官方发布的一个Node.js无头浏览器库，通过特定的API在服务端调用Chrome的一些功能。

## 大致描述
业务系统的渲染需求是大批量地生成可能高达数百页的PDF文件，如果使用Puppeteer生成这样一个PDF文件就可能需要数十秒。为了提高渲染服务的效率，需要采用多台服务器同时进行渲染。

我将渲染服务拆分为两部分：主服务和渲染服务。主服务负责将业务系统传过来的任务拆分成一个个更小维度的子任务，并将子任务放到redis的队列中，等待所有子任务完成后进行合并和上传操作；子服务定时从redis队列中获取子任务，渲染出PDF文件并上传至文件系统供主任务下载。

为了提高服务的可用性，我基于**Egg.js**进行了这个渲染服务的开发，利用Egg.js提供的Worker，可以自动根据服务器的CPU数量启动相应的渲染服务。我在测试时使用了两台六核的服务器（也就是启动了12个渲染服务），平均每小时可生成3000-4000个PDF文件（5页左右）。

## 具体实现
### 任务分发
主服务在接收到新增任务的请求后，会拆分出一系列的子任务，每个子任务都会有一个ID。为了便于管理，我为子任务的ID设置了一个规则：${主任务ID}-${报表ID}-${索引Index}。
然后将所有子任务Id放进**redis**中的**List**进行存储，同时通过判断队列的长度启动相应数量的渲染服务进行处理。

后来为了记录子任务的执行状态，又将子任务ID与状态以Key - Value的形式放进redis的一个Hash表里面，子任务的状态有四种，分别是 **-1：失败**、**0：未渲染**、**1：渲染中**、**2：渲染完成** 。

渲染服务定时从redis的队列中取出子任务，在修改子任务的状态为1后，开始渲染PDF文件。当渲染完成并上传至文件系统后，再将子任务的状态修改为2。如果中途有发生异常或PDF文件生成失败，则将子任务的状态修改为-1。

### 任务后处理
主服务定时从**redis**的**Hash**表中获取子任务的状态（目前的设定是1分钟1次），以统计任务的渲染进度。当主任务下属的所有子任务的状态都处于已完成时（-1，2），就开始执行任务后处理流程。
1. 从文件系统下载渲染好的PDF文件
2. 将对应的多个PDF文件合并成一个PDF文件（使用**easy-pdf-merge**这个库）
3. 将PDF文件打包成ZIP压缩包（使用**archiver**这个库）
4. 将压缩包上传至文件系统，并标记任务完成
5. 删除redis中的数据
6. 删除主服务器上的剩余文件以及渲染服务上传到文件系统的文件。

### 异常处理
在服务运行过程中，免不了出现**Bug或者各种意外情况**；经过测试发现，大多数问题都出现在**渲染服务**上，例如：
* 程序错误，执行渲染流程时发生错误，可以通过异常捕获将子任务标记为失败状态。
* 渲染服务被意外关闭，导致正在渲染的子任务状态一直处于渲染中（其实已经渲染失败）。
* 子任务ID从队列被取出后，没有开始渲染任务，导致状态一直处于未渲染。

对于上述第一种异常，主服务是将其视作已完成的，但是不会下载这个子任务的PDF文件，而是将其添加到异常记录中。

第二和第三种异常，由于没有进行异常的标记，所以需要进行特别的处理。我在主服务的定时任务中，会对比正在执行的主任务前后两次的渲染进度，如果进度没有发生变化，那么这个主任务就变成了**停滞状态**（因为渲染单个子任务的速度很快，在主任务两次定时任务中间一定会执行多个的，如果进度没变化就表明有部分子任务是出现异常情况了）。对于停滞状态的主任务，会获取其状态为0和1的子任务，并将这些子任务标记为失败，同样的也把他们添加到异常记录中，在主任务一下轮更新后即可开始任务后处理。

主任务在执行完任务后处理后，如果有被标记为失败的子任务，则会自动调用一个数据处理的接口，用于重新添加一个特殊的主任务用于重新渲染这些失败的子任务。

### 优化执行流程
**Puppeteer**官方文档的渲染PDF流程是这样的：

打开浏览器 -> 新建Page(Tab页) -> 跳转至指定URL -> 等待页面渲染完成 -> 生成PDF文件 -> 关闭Page -> 关闭浏览器

而真正服务于PDF文件渲染的只是 新建Page(Tab页) -> 跳转至指定URL -> 等待页面渲染完成 -> 生成PDF文件 -> 关闭Page 这中间的5个流程。那么要优化这个执行流程也比较简单，就是在渲染服务启动后就自动启动浏览器，并将**浏览器的节点**存储到内存中，当需要执行渲染任务时通过wsEndpoint方法**重新连接浏览器**即可。当渲染服务关闭前，自动关闭浏览器即可。
通过复用浏览器可以节省一部分加载/关闭浏览器的时间，但是开启关闭的Page达到一定数量后，有可能会导致浏览器崩溃（因为页面阻塞或内存泄漏），所以需要定时**重启浏览器**。



以下是对**Puppeteer**进行封装的代码
```javascript
const puppeteer = require('puppeteer')
// 浏览器最大连接计数(超过后需重启)
const BROWSER_MAX_COUNT = 1000

let store = {
  // 根据ID保存浏览器节点
  browsers: {}
}

/**
 * 浏览器连接工具
 */
class BrowserUtil {
  /**
   * 启动浏览器
   * @returns {Promise<void>}
   */
  static async init (id) {
    // chromium启动配置 https://peter.sh/experiments/chromium-command-line-switches/
    const browser = await puppeteer.launch({
      headless:true,
      args: [
        '--disable-gpu',
        '--disable-dev-shm-usage',
        '--disable-setuid-sandbox',
        '--no-first-run',
        '--no-sandbox',
        '--no-zygote'
      ]
    })
    // 保存浏览器节点
    const endpoint = await browser.wsEndpoint()
    store.browsers[id] = {
      endpoint,
      count: 0
    }
    console.log(`初始化浏览器:${endpoint}`)
    return endpoint
  }

  /**
   * 重启浏览器
   * @returns {Promise<void>}
   */
  static async restart (id) {
    // 先关闭浏览器
    await BrowserUtil.close(id)
    // 重新启动浏览器
    await BrowserUtil.init(id)
    console.log('重启浏览器完成')
  }

  /**
   * 连接至浏览器
   * @returns {Promise<*>} 返回浏览器实例
   */
  static async connect (id = 'default') {
    let endpoint = store.browsers[id] ? store.browsers[id].endpoint : null
    // 初始化浏览器
    if (!endpoint) {
      endpoint = await BrowserUtil.init(id)
    }
    store.browsers[id].count++
    // console.log(`浏览器连接计数:${store.count}`)
    console.log(`连接到浏览器:${endpoint}`)
    return await puppeteer.connect({
      browserWSEndpoint: endpoint
    })
  }

  /**
   * 关闭浏览器
   * @returns {Promise<*>}
   */
  static async close (id) {
    // 连接当前浏览器
    const browser = await puppeteer.connect({
      browserWSEndpoint: store.browsers[id].endpoint
    })
    // 关闭当前浏览器
    await browser.close()
    // 清空store中的节点
    delete store.browsers[id]
  }

  /**
   * 检查是否需要重启浏览器
   * @returns {Promise<void>}
   */
  static async check (id = 'default') {
    if (store.browsers[id] && store.browsers[id].count > BROWSER_MAX_COUNT) {
      console.log('需要重启浏览器')
      await BrowserUtil.restart(id)
    }
  }
}

module.exports = BrowserUtil
```

## 结束
由于很长时间没有写技术相关的文章了，所以上文描述可能不是很清晰，等我有空也会整理好思路补充完整，感谢各位的阅读。
