---
title: "Node"
date: 2020-10-15T01:16:54+08:00
draft: true
indent: true
indentFirstParagraph: true
comments: true
toc: false
tags: ["Node.js"]
---



### Isolate

`Isolate`代表一个独立的`V8`实例，不同的`Isolate`之间是相互隔离的，不共享资源、状态，并且拥有各自的内存堆。一个计算机线程可以处理多个`Isolate`，但同一时间`Isolate`只能由一个线程处理，简而言之，`Isolate`是一个`V8`虚拟机，可以由不同的线程在不同的时间执行。

### Handle



## 事件循环的流程

事件循环，event loop，实现于libuv

- 观察者
- 非I/O的异步API
  - process.nextTick()和setImmediate()区别

## 异步I/O的流程

## 微任务、宏任务

## Promise

## 全局对象

## 内存管理

## 垃圾回收

## 内存泄漏

## Cookie、Session

## 多进程架构

- spawn() fork() exec() execFile() 区别
- stdio
- 进程间通信

## WebSocket

## HTTPS原理

