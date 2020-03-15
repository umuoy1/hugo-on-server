---
title: "Cookie和Session"
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

### 分类

这里直接引用维基百科的介绍：

Cookie总是保存在客户端中，按在客户端中的存储位置，可分为内存Cookie和硬盘Cookie。

内存Cookie由浏览器维护，保存在内存中，浏览器关闭后就消失了，其存在时间是短暂的。硬盘Cookie保存在硬盘裡，有一个过期时间，除非用户手工清理或到了过期时间，硬盘Cookie不会被删除，其存在时间是长期的。所以，按存在时间，可分为非持久Cookie和持久Cookie

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

1. cookie 在 http 请求中是鸣文传递的，有信息泄露的风险
2. cookie 的大小有限制，不易存储复杂信息



------



## Session

session 的本意是会话状态，在 http 中的实现是用来跟踪用户的状态。

由于HTTP协议是无状态的，服务端不知道用户上一次做了什么，所以服务端会在用户初始状态的时候创建特定的 session ，用来标识并跟踪这个用户。

### 原理

session 是保存在服务端中的，第一次创建 session 时，服务端会创建一个特殊的表示 session ID，并且通过 cookie 返回给客户端，客户端将含有 session ID 的cookie保存。之后客户端在访问服务端时，就可以把保存有 session ID 的 cookie 发送给服务端，服务端就会指定客户端是谁了。

可见 session 是通过 cookie 来实现的，既然服务端可以通过 session 知道客户端的身份，用户的 账号密码等信息就可以存储在 session 里，解决了 cookie 不安全的问题。



------



## 总结

Cookie 保存在客户端中，用来存储用户的信息。

Session 保存在服务端中，用来跟踪用户，对用户进行唯一标识，可以通过 cookie 实现。

