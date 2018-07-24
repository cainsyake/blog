---
title: 通过js实现Hibernate的持久化对象复制
date: 2017-12-14 14:21:23
tags:
---

整体思路
---
1. 通过Ajax获取要复制的对象 srcObj
2. 遍历srcObj，将其中全部id属性(也就是主键)delete掉
3. 通过Ajax提交处理后的srcObj，后端直接save(srcObj)即可保存为新的对象。

（如果srcObj是在页面js的全局变量中获取的，需要进行深拷贝，然后用拷贝后的对象进行上述2、3步的处理）<br>
<!-- more -->

实现代码
---
js测试样例(这个测试复制的对象是js的一个全局变量，所以需要进行深拷贝后再作其余操作)
```javascript
function cloneObject(){
    var testObj = deepCopy(rule);   //rule是全局变量，也是被复制的对象
    console.log('输出处理前对象');
    console.log(testObj);
    console.log('开始遍历对象');
    eachProps(testObj);
    console.log('结束遍历对象,输出处理后对象');
    console.log(testObj);
    console.log('检查rule对象');
    console.log(rule);
    var ajaxData = {rule: testObj, username: 'cloneTest'};
    $.ajax({
        url: '/addRule',
        type: 'POST',
        contentType: "application/json",
        data: JSON.stringify(ajaxData),
        dataType:'json',
        success:function (rs) {
            console.log('Clone Success');
        },
        error:function (rs) {
            console.log(rs);
        }
    });
}


function deepCopy(obj) {
    var str, newobj = obj.constructor === Array ? [] : {};
    if(typeof obj !== 'object'){
        return;
    } else if(window.JSON){
        str = JSON.stringify(obj); //系列化对象
        newobj = JSON.parse(str); //还原
    } else {
        for(var i in obj){
            newobj[i] = typeof obj[i] === 'object' ?
                cloneObj(obj[i]) : obj[i];
        }
    }
    return newobj;
}

function eachProps(obj) {

    for (var prop in obj) {
        if (typeof (obj[prop]) == 'object') {
            eachProps(obj[prop]);
        } else if (typeof (obj[prop]) == 'array') {
            obj[prop].forEach(function (subObj, index, array) {
                eachProps(subObj);
            });
        } else if (prop === 'id') {
            // console.log('捕捉到id:' + obj[prop]);
            delete obj[prop];
            // console.log(obj);
            // console.log('-----');

        }
    }

}
```

实现原理：
关于[Hibernate持久化对象的介绍](http://blog.csdn.net/yyywyr/article/details/6645040)
在上面的实现代码里，eachProps(obj)这个函数将对象的全部id(主键)delete掉，再交给后端JPA进行save操作时，会被当作一个新的对象(SQL: insert操作)。
如果没有把对象的主键delete掉，JPA执行的将会时update操作，从而导致异常。