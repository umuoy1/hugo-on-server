---
title: "Cookie，Session和token"
date: 2020-03-15T17:42:40+08:00
draft: false
indent: true
indentFirstParagraph: true
comments: true
toc: true
tags: []
---

## Cookie

### 定义

Cookie，中文名「小甜饼」，是一段用来标识用户身份和附加信息而存储在本地客户端上的字符串。

### 特点

Cookie有以下几个特点

- 大小有限制，一般不超过5kb
- cookie 是没有结构的，但可以用格式如 "a1=v1;a2=v2;a3=v3"来存储结构化数据
- 跨域不共享，即浏览器为每个域名存储一个 cookie，
- 浏览器每次发送 http 请求，都会将请求域的 cookie 发送给服务端。这里的请求域仅指本次请求的域，比如在本站首页向 网易云音乐 发送 http 请求，则会向 网易云云音乐 发送本地存储的 cookie 。
- 服务端可以修改 cookie 并返回给浏览器。
- 浏览器可以在限制条件下修改 cookie 并返回给服务端。
- cookie 在指定的域名及根域名下都生效。

### 使用

以下的例子在 Node.js 中演示。

首先是解析 cookie

```javascript
req.cookie = {}
    const cookieStr = req.headers.cookie || ''
    cookieStr.split(';').forEach(element => {
        if(!element){
            return
        }
        const arr = element.split('=')
        const key = arr[0]
        const val = arr[1]
        req.cookie[key]=val
    });
    console.log(req.cookie) //为了方便测试，把cookie打印出来
```

在有登录验证的场景中，可以使用 cookie 验证用户是否登录

```javascript
if(req.cookie.username) {
            //已登录
        }
        else {
            //尚未登陆
        }
```

服务端设置 cookie

```javascript
//假如已经登录成功，设置 cookie 返回给客户端
res.setHeader('Set-Cookie',`usernamed=${userNamed}; path=/`)
```



刚才提到客户端可以修改 cookie 并返回给服务端，但修改是有限制的，假如客户端能修改 cookie 来绕过登录验证，是很危险的,可以添加 `httpOnly` 属性来允许 cookie 仅能由服务端修改，这里的修改是指不能对已设置的 cookie 修改和删除，但可以追加。

```javascript
res.setHeader('Set-Cookie',`usernamed=${userNamed}; path=/; httpOnly`)
```



在很多网站的登录页面，都可以看到类似"保存用户名xx天"的选项，实现原理就是修改 cookie 的过期时间，这里设置过期时间为 1day。

```javascript
const getCookieExpires = () => {
    const d = new Date()
    d.setTime(d.getTime() + (24 * 60 * 60 * 1000))
    return d.toGMTString() //cookie 中的expires为GMT格式
}

res.setHeader('Set-Cookie', `usernamed=${userNamed}; httpOnly; expires=${getCookieExpires()}`)

```

### 缺陷

cookie 通常有以下几个缺陷

1. cookie 在 http 请求中是明文传递的，有信息泄露的风险
2. cookie 的大小有限制，不易存储复杂信息

------



## Session

session 的本意是会话状态，在 http 中的实现是用来跟踪用户的状态。

由于HTTP协议是无状态的，服务端不知道用户上一次做了什么，所以服务端会在用户初始状态的时候创建特定的 session ，用来标识并跟踪这个用户。

就比如购物网站的购物车，我添加了商品，服务端也做了相应的处理，但服务端不知道这个商品添加到哪，也不知道这个购物车是谁了，有了session，客户端就可以通过它来追踪我，然后服务端也就这一系列行为是我做的。

### 原理

1. 服务端在 cookie 中找到对应的seesionid。
2. 根据 sessionid，从服务端对应的 session 中获取数据，然后返回。
3. 如果找不到 sessionid，服务端创建一个新 session，并将 session 添加到 cookie 中，写入响应头。

### 缺点

session 保证了消息的可追踪，也保证了安全性，但也有问题，如果服务器做了负载均衡，我刚才的请求是 A 机器处理的，但如果我下一个请求到了 B 机器，我的 session 又还存在A机器上，B机器无法根据传过来的 sessionid 确定我是谁，只能让我重新登陆。

有解决方法是把 session 单独存到一台服务器上，对其他 Web 机器共享。

------

## Token

### 原理

1. 客户端登录，服务端验证用户名和密码之后，对用户名编码，生成 token 返回给客户端。
2. 客户端将 token 储存，并在每次发送请求的时候附上 token。
3. 客户端后续发送请求，服务端对用户名用同样的方式编码，然后和客户端发过来的 token 做对比，以验证用户合法性。

### 特点

1. Token 不依赖于 Cookie ，可以在不支持 Cookie 的平台上使用，也不会有CSFR攻击的风险
2. Token 在服务端生成和验证
3. Token 储存在客户端，比如存在 Cookie ， sessionStorage ， localStorage 中。