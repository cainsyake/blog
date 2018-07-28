---
title: '逃离回调地狱,从Promise到Async/Await'
date: 2018-07-28 10:52:53
tags:
---
在JavaScript中异步编程是非常常见的，从Ajax请求到文件读写，通过异步我们能更高效的利用计算资源。随着JavaScript的快速发展，异步编程的方式也在不断发展变化，从callback到ES6的Promise，再到ES7的Async/Await。开发者要进行JavaScript进行异步编程也是变得越来越简便。
<!-- more -->
### callback
在我最开始是通过Ajax接触异步的，那时候我还没有很完整的去学习JavaScript，所以写出来的代码是这样子的。
```javascript
// 回调函数
function success (data) {
    console.log(data) // do something
}
// 发起Ajax请求
function sendRequest (callback) {
    $.get("http://localhost:8080/api", function(data,status){
        callback(data)  
    })
}
sendRequest(success)    // 将回调函数以参数的方式传给发起Ajax请求的函数
```
为了方便描述，我在上面的例子中将回调函数(success)拆分出来。当遇到一些复杂的业务需要依次发起多次请求时，就可能造成多层回调嵌套，给代码的维护带来很大的困难。例如一个业务的执行流程是：<br>
发起请求A ==> 根据A的结果发起请求B ==> 根据B的结果发起请求C ==> ...<br>
它的代码可能是这样子的
```javascript
$.get("http://localhost:8080/a", function(dataA,statusA){
    $.post("http://localhost:8080/b", dataA, function(dataB,statusB){
        $.post("http://localhost:8080/c", dataB, function(dataC,statusC){
            console.log(dataC)  // 或许还有更多的嵌套
        })
    })
})
```
---
### Promise
还好，ES6给我们带来了Promise,所谓Promise，字面上可以理解为“承诺”，就是说A调用B，B返回一个“承诺”给A，然后A就可以在写计划的时候这么写：当B返回结果给我的时候，A执行方案Plan1，反之如果B因为什么原因没有给到A想要的结果，那么A执行应急方案Plan2，这样一来，所有可能出现的情况都在A的掌控之中了。<br>
（关于Promise的学习，我当时是在 [Promise - 廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000/0014345008539155e93fc16046d4bb7854943814c4f9dc2000) 开始了解Promise的）<br>
通过**Promise的链式调用**，我们可以把上面例子的代码改造成下面这样：
```javascript
axios.get('http://localhost:8080/a').then(data => {
    return axios.post('http://localhost:8080/b', data) // 将请求A返回的data作为请求B的requestData
}).then(data => {
    return axios.post('http://localhost:8080/c', data) // 将请求B返回的data作为请求C的requestData
}).then(data => {
    console.log(data)   // do something
}).catch(error => {
    // 在Promise链上发生的异常都会在这里被捕获
    // 我们应该对错误/异常进行处理
    console.log(error)
})
```
在then方法中只要**返回一个Promise对象**，then方法就会继续调用。如果在Promise链上发生了错误/异常，就会被最后面的**catch**捕获到。<br><br>

---
### Async和Await
也许我们会想，做到这样就可以了吧？哈哈，飞速更新发展的前端技术让许多程序猿不得不感叹:别更新了，我都快学不动了。<br>
ES7出来后，Async/Await这个异步特性能让异步编程更加优雅。
关于Async和Await，可以先通过这篇文章进行一个基本的了解：[深入理解 ES7 的 async/await](https://juejin.im/entry/58523b908e450a006c4d0c5b)。<br>
还是回到实践上吧，下面是前几天写的一个项目里面的代码。
```javascript
/**
 * 一些说明：
 * FileUtil的modify方法是基于Promise的异步方法，用于修改一个File对象的属性
 * addFileAuthUser方法同样也是基于Promise的异步方法，通过axios发送请求
 */

// 这个是基于ES6 Promise链式调用的写法
handleAddFileAuthUser({ commit, state }, params) {
    return new Promise((resolve, reject) => {
        const { file } = params
        const fileUtil = new FileUtil(file, state.rootFile)
        addFileAuthUser(params).then(res => {
            if (res.state === 0) {
                return fileUtil.modify()
            } else {
                reject(res)
            }
        }).then(res => {
            commit('setRootFile', res)
            resolve()
        }).catch(error => {
            reject(error)
        })
    })
}

// 这个是基于ES7 Async/Await的写法
handleAddFileAuthUser({ commit, state }, params) {
    return new Promise((resolve, reject) => {
        const { file } = params
        const fileUtil = new FileUtil(file, state.rootFile)
        const action = async () => {
            try {
                const apiResult = await addFileAuthUser(params)
                if (apiResult.state === 0) {
                    const modifyResult = await fileUtil.modify()
                    commit('setRootFile', modifyResult)
                    resolve()
                } else {
                    reject(apiResult)
                }
            } catch (error) {
                reject(error)
            }
        }
        action()
    })
}
```
先谈一下我个人的感受，采用ES7 Async/Await的写法，可以让异步代码看起来更像是同步代码，代码逻辑比Promise链式调用的写法更加清晰。不过，Async/Await的实现仍然离不开Promise，因此我们在学习它之前还是要掌握好Promise。同时，Async/Await只是写法看起来像是同步的，但它本质上还是异步的，因此我们不必过于担心它会阻塞我们的JavaScript运行。<br>
关于Async/Await,下面几篇文章也会是很好的参考：
* [理解 async/await](https://segmentfault.com/a/1190000010244279)
* [ES6/7/8新特性Promise,async,await,fetch带我们逃离异步回调的深渊](https://blog.csdn.net/wang839305939/article/details/75505444/)

以上是我这个前端小菜鸟对ES6 Promise、ES7 Async/Await的一些个人理解，写的不好的地方还有请各位读者多多指正，谢谢！