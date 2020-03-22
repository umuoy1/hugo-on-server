---
title: "Golang入门总结"
date: 2020-03-08T01:35:09+08:00
draft: false
comments: true
toc: true
tags: ["Go"]
---

## 输入输出

Golang 有类似C语言的格式化输入输出，常用占位符如下：

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

Golang中的变量一经定义，必须使用

全局定义的变量，如果首字母大写则对其他包可以访问。

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

 一个字符串是一个不可改变的字符序列。 字符串的任何操作会产生一个新字符串并返回。

字符串可以用反引号代替双引号，表示字符串的原生格式。

```
var s string = `
Line: 1
Line: 2  //这是第二行`

Line: 1
Line: 2  //这是第二行 
```

字符串的底层是一个`byte`的数组，也可以进行切片操作

```
s1 := "abcd"
s2 := s1[1:3]//bc
```

可以通过`string`和`rune`的类型转换，达到改变字符串的目的

```
s1 := "abcd"
s2 := []rune(s1)
s2[1]='B'
s1 = string(s2)
fmt.Println(s1)//"aBcd"
```

为何不用`[]byte`？

考虑到中文字符



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

 切片(slice)是在数组之上的抽象数据类型，数组不够灵活，切片应运而生。

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

#### append, copy

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

切片可以通过`copy()`函数拷贝

```
a := []int{1,2,3,4,5}
b := make([]int,10)
copy(b,a)
```

`copy`为值拷贝，改变原切片的值不会影响新切片

如果`copy`的目标切片的cap比原切片的len小时，忽略后面的值

```
a := []int{1,2,3,4,5}
b := make([]int,2)
copy(b,a)
fmt.Println(b)
//[1 2]
```



## 函数

在Golang中，函数是一等公民。

#### 函数的声明

```
func func_name(args...) (return-type){
	body
}
```

#### 参数传递

Golang中，无论是值传递还是引用传递，传递给函数的都是变量的副本，值传递为值的拷贝，引用传递为地址的拷贝。

其中`map , slice, chan, 指针, interface`默认为引用传递。

#### 可变参数

args可以是不定长的

```
func add(arg...)int{
	...
}
```

传入参数保存在`arg`中，arg是一个slice。使用`arg[i]`访问参数，`len(arg)`获取参数个数。

#### defer

`defer`关键字表示的语句，会在函数返回时执行，`defer`语句中的变量，在`defer`声明时就决定了

多个`defer`以栈的顺序执行。

```
func f(){
	i := 3
	refer fmt.Println(i)
	i++
	refer fmt.Println(i)
	return
}
//4
//3
```

---

## 常用内置函数

#### new

`new`给变量分配内存，并置为零值，然后返回指针

```
i := new(int)
//(*i)==0
s := new(string)
//(*s)==""
```

#### make

`make`用于给引用类型分配内存并初始化，如`slice, map, channel`。

```
v := make([]int,10,50)//len=10, cap=50
c := make(chan int,10)
```

---

## 结构体

Golang 中没有类的概念，但结构体可以看作是其他语言的类。

### 声明

```go
type 标识符 struct{
	field1 type
	field2 type
}
```

### 定义

结构体的定义有三种形式

```go
var A People
var A *People  = new(People)
var A *People = &People{}
```

### 访问

和C语言一样，使用`.`

```
type People struct{
	Name string
	age int
}

fmt.println(A.Name)//指针形式也可以通过此方式访问
fmt.println((*A).Name)
```

### 初始化

```go
A := People{Name:"Youmu",}
```

Golang没有构造函数，可以通过工厂模式实现结构体的实例化。

```go
func NewPeople(name string,age ing)(*People) {
	return &People{
		Name: name,
		Age:  age,
	}
}

people := NewPeople("Youmu",14)
```

### tag

结构体的每个字段可以写上一个`tag`，这个`tag`可以在运行时通过`reflection`包读取。`tag`通常用在`json`的转换中。

```go
type People struct{
	Name string `json:"user_name"`
	age int
}

A := People(Name:"Youmu",age:14)
d,_ := json.Marshal(A)
fmt.Println(string(A))
//{"user_name":"Youmu"}
//age为首字母小写，外部包"encoding/json"对其不可访问。
```

### 匿名字段 和 继承

匿名字段通常搭配继承使用

```
type People struct{
	Name string
	age int
}

type Student struct{
	People	//Studeng继承People的所有属性，方法
	score int
}

stu := Student{
	People{
		Name: "Marisa"
		age:  15
	},
	score:	0,
}
fmt.Println(stu.Name,stu.People.Name,stu.age,stu.score)
//可以隐式地引用所继承结构体的变量，但是如果结构体中有重名变量，需要显示引用
```

### 方法

```
type People struct{
	Name string
	age int
}
func (this People)eat()(string){//该方法属于结构体People
	return this.Name + " eat something"
}
func (this *People)setAge(age int){
	this.Age = age
}
A := People(Name:"Yuyuko",age:1000)
A.setAge(1001)
fmt.Println(A.eat(),A.Age)
//Yuyuko eat something 1001
```



## goroute



## channel

-

## 异常处理

panic

-

## time

-

## 闭包

