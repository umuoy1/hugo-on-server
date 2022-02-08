---
title: "JavaScript 正则表达式 test 方法在 V8 中的执行路径"
date: 2022-02-06T01:57:27+08:00
draft: false
indent: true
indentFirstParagraph: true
comments: true
toc: false
tags: []
---

`COMMIT ID:`[c97337ff5ece3c4838fedebddd762984d90ba6f4](https://github.com/v8/v8/tree/c97337ff5ece3c4838fedebddd762984d90ba6f4)

## regexp.prototype.test

`regexp-test.tq`
// 快速模式

```
-> transitioning javascript builtin RegExpPrototypeTest -> RegExpPrototypeExecBodyWithoutResultFast
=> regexp.tq
-> transitioning macro RegExpPrototypeExecBodyWithoutResultFast -> RegExpPrototypeExecBodyWithoutResult -> transitioning macro RegExpPrototypeExecBodyWithoutResult -> RegExpExecInternal
=> builtins-regexp-gen.cc
-> TNode<HeapObject> RegExpBuiltinsAssembler::RegExpExecInternal
-> Runtime::kRegExpExec
=> runtime-regexp.cc
-> RUNTIME_FUNCTION(Runtime_RegExpExec) -> RegExpExec
-> MaybeHandle<Object> RegExpExec -> RegExp::Exec
=> regexp.cc
-> MaybeHandle<Object> RegExp::Exec
-> RegExpImpl::IrregexpExec
```



// 慢速模式

```
-> transitioning javascript builtin RegExpPrototypeTest -> RegExpExec
=> regexp.tq
-> transitioning macro RegExpExec(implicit context: Context) -> RegExpPrototypeExecSlow
=> regexp-exec.tq
-> RegExpPrototypeExecSlow(implicit context: Context) -> RegExpPrototypeExecBodySlow -> RegExpPrototypeExecBody
=> regexp.tq
-> transitioning macro RegExpPrototypeExecBody(implicit context: Context) -> RegExpPrototypeExecBodyWithoutResult -> transitioning macro RegExpPrototypeExecBodyWithoutResult -> RegExpExecInternal
=> builtins-regexp-gen.cc
-> TNode<HeapObject> RegExpBuiltinsAssembler::RegExpExecInternal
-> Runtime::kRegExpExec
=> runtime-regexp.cc
-> RUNTIME_FUNCTION(Runtime_RegExpExec) -> RegExpExec
-> MaybeHandle<Object> RegExpExec -> RegExp::Exec
=> regexp.cc
-> MaybeHandle<Object> RegExp::Exec
-> RegExpImpl::IrregexpExec
```