---
title: "V8 Flatten"
date: 2022-02-03T01:49:27+08:00
draft: true
indent: true
indentFirstParagraph: true
comments: true
toc: false
tags: []
---

字符串在 V8 中的表达可以概括为两种：

1. 分配在堆上的字符串数组。

 	2. 采用 [Rope](https://en.wikipedia.org/wiki/Rope_(data_structure)) 的树状结构，用来表达连接的字符串。字符串连接时，采用树状的连接方式会比申请一个新的字符串数组的消耗要小。

使用树状的结构会节省字符串拼接时的消耗，但在一些场景下比如大量字符串拼接，复杂度会变高，V8 会将多层的树状结构扁平化为字符串数组，以提高访问效率，但调用时机不由开发者指定，而是在 V8 引擎内部。

关于 v8 flatten string 更详细的介绍可以看 [奇技淫巧学 V8 之七，字符串的扁平化 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/28907384) ，这里不做赘述。

上面提到 v8 flatten 触发的时机不由开发者控制，但有一些方式间接地调用v8的`String::Flatten`。

Flatten 会显著地提高性能，但会造成原string的复制，增加堆内存的使用，不正当的频繁使用也会造成性能障碍。

flatstr 是一个可以使字符串扁平化的库，库的核心代码非常简短，最新版本只有四行

```javascript
function flatstr (s) {
  s | 0
  return s
}
```

随着nodejs以及v8版本的变更，这种hack的方式也在更新，按照commit号(正序)进行介绍：

//TODO benchmark 最优的方案 为什么最优

一、正则表达式

```javascript
var rx = /()/
function alt1 (s) {
  rx.test(s)
  return s
}
function alt2 (s) {
  rx.exec(s)
  return s
}
```

v8的正则执行路径有fast和slow两种，但最终都会执行`regexp.cc`下的`MaybeHandle<Object> RegExp::Exec`函数，最后使用` irregexp `的实现`MaybeHandle<Object> RegExpImpl::IrregexpExec`，在该函数体的第二行可以看到`subject = String::Flatten(isolate, subject);`，执行字符串的扁平化。

二、与 0 相或









相关讨论： https://github.com/harttle/liquidjs/issues/161

