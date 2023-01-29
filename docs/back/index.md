---
title: 后端技术
nav:
  title: go
---
## 基础语法
### 数组
```go
//确定长度
balance := [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}

//不确定长度
balance := [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}

//  将索引为 1 和 3 的元素初始化
balance := [5]float32{1:2.0,3:7.0}
```
### 指针

一个指针变量指向了一个值的内存地址。

- 指针类型用于传递地址, 而不是传递值, 因为 golang 的函数, 所有的参数都是传递一个复制的值. 如果值的体积过大, 那么就会严重降低效率, 而传递一个地址, 就会大大提高效率. 另外传递指针也能让 go 函数实现对变量值的修改.
- 如果一个复杂类型的值被传递了若干次后, 和自己比较, 虽然用于保存的容器和名称变了, 但用于保存值的地址不变, 这个时候, 只要使用指针进行对比, 就知道还是原来的东西.

### 结构体

- 结构体可以直接从指针的内存地址去读取对应的属性
- type 和 struct 是Go语言的关键字，二者组合代表定义一个结构体。
- 结构体是聚合类型。

```go
type Books struct {
   title string
   author string
   subject string
   book_id int
}
func main() {
    var Book1 Books        /* 声明 Book1 为 Books 类型 */

       /* book 1 描述 */
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407

   :Book2 = &Book1
	fmt.Println(Book2)
	fmt.Println(Book1)
   
}
```
### 切片（slice）

Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。

```go
slice1 := make([]type, len,capacity) //make创建切片 len为长度、capacity为容量为可选参数

len(slice1) // 获取长度
cap(slice1) // 获取容量
append(slice1，1) //追加新元素
copy(slice2,slice1) //拷贝1的内容到2
```

### 函数

- 函数名称首字母小写代表私有函数，只有在同一个包中才可以被调用。
- 函数名称首字母大写，代表公有函数，不同的包也可以调用。
- 任何函数都会从属于一个包。

### 方法
```go
type Age unit

func (age Age) String(){
    fmt.Println("the age is",age)
}

var age:Age = "123"

age.String()

//String 为Age类型的方法，可以直接通过 .的形式进行调用
```
- 方法可以接收者可以为值类型和指针类型。
- 如果接收者定义为值类型，则对值的修改是无效的。
- 如果接收者定义为指针类型，则对值的修改是有效的。
- 如果使用一个值类型变量调用指针类型接受者的方法，Go语言编译器会自动帮我们取指针调用。相反也会处理。

### 范围（range）

range 关键字用于for循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素。在数组和切片中它返回元素的索引值，在集合中返回 key-value 对的 key 值。
### Map
```go
package main

import "fmt"

func main() {   
   /* 创建 map */
   countryCapitalMap := map[string] string {"France":"Paris","Italy":"Rome","Japan":"Tokyo","India":"New Delhi"}
   
   fmt.Println("原始 map")   
   
   /* 打印 map */
   for country := range countryCapitalMap {
      fmt.Println("Capital of",country,"is",countryCapitalMap[country])
   }
   
   /* 删除元素 */
   delete(countryCapitalMap,"France");
   fmt.Println("Entry for France is deleted")  
   
   fmt.Println("删除元素后 map")   
   
   /* 打印 map */
   for country := range countryCapitalMap {
      fmt.Println("Capital of",country,"is",countryCapitalMap[country])
   }
}
```

### 反射
1. 大量使用反射的代码通常会变得难以理解
2. 反射的性能低下，基于反射的代码会比正常的代码运行速度慢一到两个数量级

### 并发

并发：同一时间段内执行多个任务（你早上在编程狮学习Java和Python）

并行：同一时刻执行多个任务（你和你的网友早上都在使用编程狮学习Go）

Go语言中的并发程序主要是通过基于CSP（communicating sequential processes）的goroutine和channel来实现，当然也支持使用传统的多线程共享内存的并发方式

### goruntine

```go
package main

import (
  "fmt"
  "time"
  "sync"
)

var wg sync.WaitGroup
func hello(){
  fmt.Println("hello")
	defer wg.Done()//把计算器-1
}

func main(){
	wg.Add(1)//把计数器+1
	go hello()
	fmt.Println("hello world！")
	wg.Wait()
	// time.Sleep(time.Second)
}
```

### channel
```go
import (
	"fmt"
	"time"
)
func main(){
  a := make(chan int, 1)
	go receive(a)
	a <- 10
	fmt.Println("发送成功1")
	a <- 20
	fmt.Println("发送成功2")
	a <- 30
	fmt.Println("发送成功3")
}


func receive(x chan int) {
	ret := <-x
	time.Sleep(time.Second)
	fmt.Println("接收成功1", ret)
	ret1 := <-x
	fmt.Println("接收成功2", ret1)
	ret2 := <-x
	fmt.Println("接收成功3", ret2)
}
```

### 反射

任何接口都由两部分组成： 接口的具体类型，以及具体类型对应的值。

reflect.Value 可以通过函数 reflect.ValueOf 获取

```go
	p := person{Name: "myName", Age: 18}
	ppv := reflect.ValueOf(&p)
	pv := reflect.ValueOf(p)
	ppv.Elem().Field(0).SetString("yourName")

	fmt.Println(ppv.Kind()) // ptr
	fmt.Println(pv.Kind())  // struct
```

### go中底层类型

```go
type Kind unit
const (
   Invalid Kind = iota
   Bool
   Int
   Int8
   Int16
   Int32
   Int64
   Uint
   Uint8
   Uint16
   Uint32
   Uint64
   Uintptr
   Float32
   Float64
   Complex64
   Complex128
   Array
   Chan
   Func
   Interface
   Map
   Ptr
   Slice
   String
   Struct
   UnsafePointer
)
```


