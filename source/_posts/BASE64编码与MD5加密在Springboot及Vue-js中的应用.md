---
title: BASE64编码与MD5加密在Springboot及Vue.js中的应用
date: 2017-12-11 12:01:25
tags:
---


前端(**Vue.js**)
---
首先安装base64.js和md5.js
```
npm install --save js-base64
npm install --save js-md5
```
<!-- more -->
<br>
在.vue文件的**script**标签中引入上面两个js
```
import md5 from 'js-md5';
let Base64 = require('js-base64').Base64;
```<br>

Base64编码
```
var newStr = Base64.encode(oldStr);     //oldStr为加密前的字符串,newStr为加密后的字符串
```
Base64解码
```
var afterStr = Base64.decode(beforeStr);    //beforeStr为解密前的字符串,afterStr为解密后的字符串
```
<br><br>
md5加密（md5摘要算法不可逆）
```
var newStr = md5(oldStr);   //oldStr为加密前的字符串,newStr为加密后的字符串
```
<br>
md5输出范例()
```
md5(''); // d41d8cd98f00b204e9800998ecf8427e
md5.hex(''); // d41d8cd98f00b204e9800998ecf8427e
md5.array(''); // [212, 29, 140, 217, 143, 0, 178, 4, 233, 128, 9, 152, 236, 248, 66, 126]
md5.digest(''); // [212, 29, 140, 217, 143, 0, 178, 4, 233, 128, 9, 152, 236, 248, 66, 126]
md5.arrayBuffer(''); // ArrayBuffer
md5.buffer(''); // ArrayBuffer, deprecated, This maybe confuse with Buffer in node.js. Please use arrayBuffer instead.
md5.base64(''); // 1B2M2Y8AsgTpgAmY7PhCfg==
```

后端(**Java Springboot**)
---
直接调用org.springframework.util这个包里面的DigestUtils和Base64Utils即可
<br>
Base64编码
```
String str64 = Base64Utils.encodeToString(strIn);   //strIn为加密前的字符串,str64为加密后的字符串
```
Base64解码
```
String afterStr = new String(Base64Utils.decodeFromString(beforeStr), "UTF-8");   //beforeStr为解密前的字符串,afterStr为解密后的字符串
```
<br><br>
md5加密
```
String newStr = DigestUtils.md5DigestAsHex(oldStr.getBytes());  //oldStr为加密前的字符串,newStr为加密后的字符串
```