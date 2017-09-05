---
layout: post
title: "web storage api"
date: 2017-09-03 22:49:03 +0800
tags: web
---


前段时间做个关于页面拨打电话的需求，其中和第三方呼叫软件之间的通信是通过JavaScript来做的。这其中就涉及到需要把上一次的请求信息保存下来，并在接收到下一个请求之后，取出保存的信息做处理。这里没有使用后端服务器开发中常用的数据库或是redis等持久化工具，而是使用了sessionStorage，正好加强下基础知识  
#为什么需要保存会话信息  
HTTP协议本身是无状态的，客户端到服务器之间的每个请求都是独立的。但是很多时候，我们是需要在浏览器和服务器之间携带一些会话信息的，如为了避免重复登录，我们会使用浏览器的记住密码，这样浏览器的cookie中会记录用户名密码，请求服务器时根据cookie中的用户名密码做认证；同时对已登录的用户，服务器端会保存session，并将sessionid存入cookie，服务器端接收请求取出sessionid查看用户是否已是登录状态，从而避免再次登录

#保存会话信息的方式有哪些  
会话信息可以保存在cookie和session中，这两个的区别是：cookie是存放在浏览器端的，并保存在浏览器端；而session是由服务端产生，并保存在服务器端，其中sessionid会被放入cookie中，浏览器每次向服务器发起请求时，会把cookie放入报文头部，而session的sessionid就保存在cookie中，服务器端接收到请求之后，取出cookie，并根据cookie中的sessionid取出服务器中的session
#实际应用
看下存储在浏览器端的网络存储技术，目前可用的有Cookie，sessionStorage,localStorage,但由于Cookie要求的数据大小有限制，且易用性不好，目前常用的就是HTML5提供的技术有localStorage和sessionStorage，这两个都会提供一个浏览器端数据存储的功能，但区别是：  
* localStorage：存储的信息会一直保存，不会过期  
* sessionStorage：每个信息的有效期是一个会话周期，每次打开一个浏览器页面都会创建一个会话，刷新页面会话信息依旧存在，但关闭页面之后，此会话结束，本次会话保存的sessionStorage信息也会随之清除  
##sessionStorage  

```
//获取sessionStorage
var sessionStorate=Window.sessionStorage;
//保存数据
 sessionStorage.setItem('key', 'value'); 
//获取数据
var data = sessionStorage.getItem('key'); 
// 删除数据
sessionStorage.removeItem('key'); 
//删除sessionStorage中的所有数据
sessionStorage.clear();
```

##localStorage

操作和sessionStorage类似

```
//取得变量
localStorage = window.localStorage;
//保存数据
localStorage.setItem('key','value');
//取得数据
var data=localStorage.getItem('value');
//删除数据
localStorate.removeItem('key');
```

#其他

网上找到的一个Cookie，sessionStorage，localStorage的对比，总结的很好  
|特性|Cookie|localStorage|sessionStorage|  
|-|-|-|-|  
|数据的生命期|可设置失效时间，默认是关闭浏览器后失效|除非被清除，否则永久保存|仅在当前会话下有效，关闭页面或浏览器后被清除|  
|存放数据大小|4K左右|一般为5MB| 一般为5MB|  
|与服务器端通信|每次都会携带在HTTP头中，如果使用cookie保存过多数据会带来性能问题|仅在客户端（即浏览器）中保存，不参与和服务器的通信| 仅在客户端（即浏览器）中保存，不参与和服务器的通信|  
|易用性|需要程序员自己封装，源生的Cookie接口不友好|源生接口可以接受，亦可再次封装来对Object和Array有更好的支持|源生接口可以接受，亦可再次封装来对Object和Array有更好的支持|  

  
#参考：  
[详说 Cookie, LocalStorage 与 SessionStorage](http://jerryzou.com/posts/cookie-and-web-storage/)  
[Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API/Using_the_Web_Storage_API)  
[Window.localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)  
[Window.sessionStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage)  

>* 同样是码字，为什么码代码很轻松，码起文档来却简直要要了我的命？？？  
>* 才发现不同的markdown引擎居然支持的格式各有差别，本地wiz笔记编辑好的文档，发上去之后格式全乱了。各种引擎试了一遍，没有一个全部支持我用的特性的，现在选了Redcarpet，默认不支持表格，等有时间再研究下它的插件吧


