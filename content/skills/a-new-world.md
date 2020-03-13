---
title: "Golang入门总结"
date: 2020-03-08T01:35:09+08:00
draft: false
comments: true

---

## 输入输出

Golang有类似C语言的格式化输入输出，常用占位符如下：

```
%v		//对应值的默认格式
```

整数：

```
%d		十进制表示
%b		二进制表示
%o		八进制表示
%x		十六进制表示
```

Golang没有无符号整型的占位符

浮点数：

```
%f		一般小数
%e		科学计数法
```

字符串：

```
%s		字符串
```

指针：

```
%p		十六进制表示，前缀0x
```

#### Print

```
func Print(a ...interface{}) (n int, err error)
```

 使用其操作数的默认格式进行格式化并写入到标准输出。 当两个连续的操作数均不为字符串时，它们之间就会添加空格。 

#### Printf

```
func Printf(format string, a ...interface{}) (n int, err error)
```

根据格式说明符写入到标准输出。

#### Println

```
func Println(a ...interface{}) (n int, err error)
```

根据默认格式格式化后写入到标准输出，格式之间添加空格，最后添加换行符。

#### Scan

```
func Scan(a ...interface{}) (n int, err error)
```

 Scan 扫描从标准输入中读取的文本，并将连续由空格分隔的值存储为连续的实参。 换行符计为空格。 

#### Scanf

```
func Scanf(format string, a ...interface{}) (n int, err error)
```

扫描从标准输入中读取的文本， 并将连续由空格分隔的值存储为连续的实参 。

#### Scanln

```
func Scanln(a ...interface{}) (n int, err error)
```

类似Scan，但它在换行符处停止扫描，且最后的条目之后必须为换行符或 EOF。 

#### Sprint

```
func Sprint(a ...interface{}) string
```

 使用其操作数的默认格式进行格式化并返回其结果字符串。 当两个连续的操作数均不为字符串时，它们之间就会添加空格。   

#### Sprintf

```
func Sprintf(format string, a ...interface{}) string
```

---

## 变量

变量类型：

#### 整数

```
(u)int  (u)int8  (u)int16  (u)int32  (u)int64
```

#### 浮点数

```
float32  float64
```

#### 字符/字符串

```
string	byte  rune(每个字符占一个长度)
```

#### 布尔型

```
bool
```

#### 常量

```
const a float32 = 3.14
const (
		A = 1
		B = 2)
```

常量的值必须是编译期间确定的数字、字符串、布尔值

#### 函数变量

```
func test(a int)int{
	return a*a
}
var f = test
var f2 = func(a int)int{
	return a*a
}//匿名函数
f(3)//9
```

#### 变量声明

```
var v type
v := value
```

---

## 字符串

 一个字符串是一个不可改变的字节序列。 字符串的相加会产生一个新字符串并返回。

字符串可以用反引号代替双引号，表示字符串的原生格式。

```
var s string = `
Line: 1
Line: 2  //这是第二行`

Line: 1
Line: 2  //这是第二行 
```



#### 常用函数

```
strings包下
Index(str,s string) int//查询字符串第一次出现的位置
LastIndex(str,s string) int//查询字符串最后一次出现的位置
Replace(str,old string,new string,n) string//替换子串old为new，次数为n,-1代表无限次
Count(str,substr string) int//查询出现次数
Repeat(s string,count int) string//生成新字符串，值为s重复count次
TrimSpace(str string) string//去除首位空格
Trim(str string,cutset string) string//去除首位字符串cutstr
Split(str string,sep string)//以sep分隔开，返回一个string切片
Join(str string,sep string) string//将切片str以sep连接，返回新字符串
```

 strconv包提供了布尔型、整型数、浮点数和对应字符串的相互转换。

```
strconv包下
x, err := strconv.Atoi("123")
y, err := strconv.ParseInt("123",10,64) //10进制，64位。默认返回int64类型
```



## 数组

数组是一个由固定长度的特定类型元素组成的序列，一个数组可以由零个或多个元素组成。

数组是固定的，不同长度的数组是不同类型，数组的长度必须是常量表达式，在编译时确定。

数组定义时，默认被初始化元素类型对应的零值。

数组的长度是...时，表示数组长度根据初始化值的个数来计算。

```
b := [3]int{1,2,3}
var b [3]int = [...]int{1,2,3}
b := [...]int{1:2,3:4}//指定索引赋值
```

数组是值类型，作为函数的参数时为值传递

两个数组的长度和类型均相同时可以被比较，否则会编译错误。

## 切片

 切片是在数组之上的抽象数据类型，数组不够灵活，切片应运而生。

```
a := []int{1,2,3}
```

#### 创建切片

基于数组创建

```
a := [5]{1,2,3,4,5}
b := a[1:3]//slice
b == &a[1]
```

新切片和原数组共享内存，当切片被扩容时，会开辟新的地址，并将原切片复制过去。

也可以使用`make`创建

```
func make([]T, len, cap) []T
```

T表示被创建的切片的元素类型， 函数 `make` 接受一个类型、一个长度和一个可选的容量参数。 调用 `make` 时，内部会分配一个数组，然后返回数组对应的切片。 

```
var s []byte
s = make([]byte,5,5)
```

容量参数被忽略时，默认为`len`

```
s := make([]byte,5)
len(s) == 5
cap(s) == 5
```

#### 切片的原理

切片本身不是数组，他表面上是数组片段。它包含了指向数组的指针，片段长度，最大容量(指向的数组长度)。

![image.png](https://i.loli.net/2020/03/13/GlBdFOgMH8VpsWm.png) 

使用make([]byte,5)创建切片后，s结构如下

![image.png](https://i.loli.net/2020/03/13/59Gvb1kq82aBZf7.png) 



切片可以通过`append`函数扩容

```
func append(s []T, x ...T) []T
```

`append`函数将x添加到切片s的尾部，并在必要时增加容量。

```
s := []int{1,2,3}
q := s
//q==s
append(q,4,5)
//q!=s
```

对达到最大容量的切片扩容时，会创建一个新的更大的切片并将原数据复制过去。

## 函数

格式

传参方式

refer

---

## 常用内置函数

new 分配内存 值类型 返回指针

make 分配内存 引用类型

---

## goroute



## channel

-

## 异常处理

panic

-

## time

-

## 闭包

