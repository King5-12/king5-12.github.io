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

结构体可以直接从指针的内存地址去读取对应的属性

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

### 范围（range）
### Map
