---
layout: Golang
title: Golang学习(面试篇)
categories:
  - Golang
tags:
  - Golang
translate_title: golang-learning-interview
date: 2019-10-31 15:46:04
---
收集了一些Golang中常见但容易入"坑"的知识点。
<!--more-->

# 考点：`defer`执行顺序

问题：写出下面代码输出内容
```go
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() {fmt.Println("打印前")}()
	defer func() {fmt.Println("打印中")}()
	defer func() {fmt.Println("打印后")}()
	panic("触发异常")
}
```
输出：
```go
打印后
打印中
打印前
panic: 触发异常

goroutine 1 [running]:
main.defer_call()
	/tmp/sandbox550202885/prog.go:15 +0xe0
main.main()
	/tmp/sandbox550202885/prog.go:8 +0x20

Program exited: status 2.
```
解析：
> defer 是后进先出。panic 需要等defer 结束后才会向上传递。出现panic的时候，会先按照defer的后入先出的顺序执行，最后才会执行panic。

拓展：
> defer、panic、recover、return的使用场景
>
> panic和recover的使用规则:
> 1. 调用panic后，调用方函数执行从当前调用点退出
> 2. 通过panic可以设定返回值,当panic函数没有被调用或者没有返回值时，recover返回Nil
> 3. recover函数只有在defer代码块中才会有效果
> 4. defer只对当前协程有效（main可以看作是主协程）
> 5. 当panic发生时依然会执行当前（主）协程中已声明的defer，但如果所有defer都未调用recover()进行异常恢复，则会在执行完所有defer后引发整个进程崩溃；
> 6. 主动调用os.Exit(int)退出进程时，已声明的defer将不再被执行。


情景1
```go
package main

import (
	"fmt"
)

func main() {
	defer func() { // 必须要先声明defer，否则不能捕获到panic异常
		fmt.Println("c")
		if err := recover(); err != nil {
			fmt.Println(err) // 这里的err其实就是panic传入的内容，55
		}
		fmt.Println("d")
	}()
	f()
	fmt.Println("e")
}

func f() {
	fmt.Println("a")
	panic(55)
	fmt.Println("b")
}
```
输出：
```go
./prog.go:21:2: unreachable code
Go vet exited.

a
c
55
d
```
分析：
> 如上规则1，调用panic后，调用方函数执行从当前调用点退出

情景2
```go
package main

import (
	"fmt"
)

func main() {
	i := 0
	defer fmt.Println(i)
	i++
	return
}
```
输出：
```go
0
```
分析：
> Go 语言中所有的函数调用其实都是值传递的，defer 虽然是一个关键字，但是也继承了这个特性

假设我们有以下的代码，在运行这段代码时会打印出 `0`：
```go
package main

import (
	"fmt"
)

type Test struct {
	value int
}

func (t Test) print() {
	fmt.Println(t.value)
}

func main() {
	test := Test{}
	defer test.print()
	test.value += 1
}
```
这其实表明当`defer`调用时其实会对函数中引用的外部参数进行拷贝，所以 test.value += 1 操作并没有修改被 defer 捕获的 test 结构体，不过如果我们修改 print 函数签名的话，其实结果就输出`1`：
```go
package main

import (
	"fmt"
)

type Test struct {
	value int
}

func (t *Test) print() {
	fmt.Println(t.value)
}

func main() {
	test := Test{}
	defer test.print()
	test.value += 1
}
```
这里再调用 defer关键字时其实也是进行的值传递，只是发生复制的是指向 test 的指针，我们可以将 test 变量理解成 print函数的第一个参数，在上一段代码中这个参数的类型是结构体，所以会复制整个结构体，而在这段代码中，拷贝的其实是指针，所以当我们修改 test.value 时，defer捕获的指针其实就能够访问到修改后的变量了。


情景3
```go
package main

import (
	"fmt"
)

func trace(s string) string {
	fmt.Println("entering:", s)
	return s
}

func un(s string) {
	fmt.Println("leaving:", s)
}

func a() {
	defer un(trace("a"))
	fmt.Println("in a")
}

func b() {
	defer un(trace("b"))
	fmt.Println("in b")
	a()
}

func main() {
	b()
}
```
输出：
```go
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```
分析：
> defer声明时会先计算确定参数的值，defer推迟执行的仅是其函数体。因此会先执行trace()

如下：
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	defer P(time.Now())
	time.Sleep(5e9)
	fmt.Println("1", time.Now())
}
func P(t time.Time) {
	fmt.Println("2", t)
	fmt.Println("3", time.Now())
}

// 输出结果：
// 1 2017-08-01 14:59:47.547597041 +0800 CST
// 2 2017-08-01 14:59:42.545136374 +0800 CST
// 3 2017-08-01 14:59:47.548833586 +0800 CST
```

情景4
```go
package main

import (
	"fmt"
)

func main() {
	{
		defer fmt.Println("defer runs")
		fmt.Println("block ends")
	}

	fmt.Println("main ends")
}
```
输出：
```go
block ends
main ends
defer runs
```
分析：
> `defer`并不是在退出当前代码块的作用域时执行的，`defer`只会在当前函数和方法返回之前被调用。

情景5
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	defer fmt.Println("in main")
	go func() {
		defer fmt.Println("in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}
```
输出：
```go
in goroutine
panic: 

goroutine 6 [running]:
main.main.func1()
	/tmp/sandbox750445541/prog.go:12 +0xa0
created by main.main
	/tmp/sandbox750445541/prog.go:10 +0xa0
```
分析：
>  Go 语言在发生 panic 时只会执行当前协程中的 `defer`函数

情景6
```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("a return:", a()) // 打印结果为 a return: 0
}

func a() int {
	var i int
	defer func() {
		i++
		fmt.Println("a defer2:", i) // 打印结果为 a defer2: 2
	}()
	defer func() {
		i++
		fmt.Println("a defer1:", i) // 打印结果为 a defer1: 1
	}()
	return i
}
```
```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("b return:", b()) // 打印结果为 b return: 2
}

func b() (i int) {
	defer func() {
		i++
		fmt.Println("b defer2:", i) // 打印结果为 b defer2: 2
	}()
	defer func() {
		i++
		fmt.Println("b defer1:", i) // 打印结果为 b defer1: 1
	}()
	return i // 或者直接 return 效果相同
}
```
分析：
> defer、return、返回值三者的执行顺序应该是：return最先给返回值赋值；接着defer开始执行一些收尾工作；最后RET指令携带返回值退出函数。



# 考点：for range
问题：以下代码有什么问题，说明原因。
```go
type student struct {
	Name string
	Age  int
}

func pase_student() {
	m := make(map[string]*student)

	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}

	for _, stu := range stus {
		m[stu.Name] = &stu
	}
}
```
解答：
> 这样的写法初学者经常会遇到的，很危险！ 与Java的foreach一样，都是使用副本的方式。所以m[stu.Name]=&stu实际上一致指向同一个指针， 最终该指针的值为遍历的最后一个struct的值拷贝。 就像想修改切片元素的属性：
```go
for _, stu := range stus {
	stu.Age = stu.Age + 10
}
```
也是不可行的。 大家可以试试打印出来：
```go
func pase_student() {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	// 错误写法
	for _, stu := range stus {
		m[stu.Name] = &stu
	}
	for k, v := range m {
		println(k, "=>", v.Name)
	}
	// 正确
	for i := 0; i < len(stus); i++ {
		m[stus[i].Name] = &stus[i]
	}
	for k, v := range m {
		println(k, "=>", v.Name)
	}
}
```

分析：
> for range创建了每个元素的副本（值拷贝），而不是直接返回每个元素的引用。每个副本的地址相同，因此value指向相同的地址。所以在for循环中使用并发就需要特别注意。因为并发，不同的goroutine可能会读取到相同的值。

```go
var wg sync.WaitGroup
var s []int = []int{1, 2, 3}
for _, value := range s {
    wg.Add(1)
    go func(){
        defer wg.Done()
        t.Logf("go routine value %d", value)
    }()
}
wg.Wait()


输出：
blog_test.go:15: go routine value 3
blog_test.go:15: go routine value 3
blog_test.go:15: go routine value 3
```
如何解决上面的问题，需要使用到值传递，重新再匿名函数中声明一个参数。

```go
var wg sync.WaitGroup
var s []int = []int{1, 2, 3}
for _, value := range s {
	wg.Add(1)
	go func(value int) {
		defer wg.Done()
		t.Logf("go routine value %d", value)
	}(value)
}
wg.Wait()
```
拓展：
```go
package main

import "fmt"

type Test struct {
	name string
}

func (this *Test) Point() { // this  为指针
	fmt.Println(this.name)
}

func main() {
	ts := []Test{{"a"}, {"b"}, {"c"}}
	for _, t := range ts {
		defer t.Point() //输出 c c c
	}
}
```
分析：
> 考察defer和方法接收者的规则，在执行t.Point()时，得到的是t的地址（引用），for结束时，t被赋值为”c“的地址，main函数返回时，都在执行”c“的接收方法Point，所以输出都是”c".

# 考点：goroutine 执行的随机性和闭包 
下面的代码会输出什么，并说明原因。
```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(1)

	wg := sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("A: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("B: ", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```
解答：
> 谁也不知道执行后打印的顺序是什么样的，所以只能说是随机数字。但是A:均为输出10，B:从0~9输出(顺序不定)。 第一个go func中i是外部for的一个变量，地址不变化。遍历完成后，最终i=10。 故go func执行时，i的值始终是10。
第二个go func中i是函数参数，与外部for中的i完全是两个变量。 尾部(i)将发生值拷贝，go func内部指向值拷贝地址。

# 考点：struct的组合继承
下面代码会输出什么？

```go
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teachershowB")
}
func main() {
	t := Teacher{}
	t.ShowA()
}
```
输出：
```
showA
showB
```
分析：
> 这是Golang的组合模式，可以实现OOP的继承。 被组合的类型People所包含的方法虽然升级成了外部类型Teacher这个组合类型的方法（一定要是匿名字段），但它们的方法(ShowA())调用时接受者并没有发生变化。 此时People类型并不知道自己会被什么类型组合，当然也就无法调用方法时去使用未知的组合者Teacher类型的功能。
 

# 考点：select随机性
下面代码会触发异常吗？请详细说明
```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	runtime.GOMAXPROCS(1)

	int_chan := make(chan int, 1)
	string_chan := make(chan string, 1)

	int_chan <- 1
	string_chan <- "hello"

	select {
	case value := <-int_chan:
		fmt.Println(value)
	case value := <-string_chan:
		panic(value)
	}
}
```
解答：
> select 是 Go 中的一个控制结构，类似于用于通信的 switch 语句。每个 case 必须是一个通信操作，要么是发送要么是接收。 
>
> select会随机选择一个可用通道做收发操作。 所以代码是有可能触发异常，也有可能不会。 
>
> 以下描述了 select 语句的语法：
- 每个 case 都必须是一个通信
- 所有 channel 表达式都会被求值
- 所有被发送的表达式都会被求值
- 如果任意某个通信可以进行，它就执行，其他被忽略。
- 如果有多个 case 都可以运行，Select 会随机公平地选出一个执行。其他不会执行。
否则：
    - 如果有 default 子句，则执行该语句。
    - 如果没有 default 子句，select 将阻塞，直到某个通信可以运行；Go 不会重新对 channel 或值进行求值。

# 考点：make默认值和append
请写出以下输入内容
```go
func main() {    
    s := make([]int,5)
    s = append(s,1, 2, 3)
    fmt.Println(s)
}
```
解答：
> make初始化是有默认值的哦，此处默认值为0
>
> [00000123]

# 考点：chan缓存池
下面的迭代会有什么问题？
```go
func (set *threadSafeSet) Iter() <-chan interface{} {
	ch := make(chan interface{})
	go func() {
		set.RLock()
		for elem := range set.s {
			ch <- elem
		}
		close(ch)
		set.RUnlock()
	}()
	return ch
}
```

解答：
> 既然是迭代就会要求set.s全部可以遍历一次。但是chan是没有缓存的，那就代表这写入一次就会阻塞。迭代可能不能完成。 

我们把代码恢复为可以运行的方式，看看效果
```go
package main

import (
	"fmt"
	"sync"
)

type threadSafeSet struct {
	sync.RWMutex
	s []interface{}
}

func (set *threadSafeSet) Iter() <-chan interface{} {
	// ch := make(chan interface{})
	ch := make(chan interface{}, len(set.s))
	go func() {
		set.RLock()
		for elem, value := range set.s {
			ch <- value
			println("Iter:", elem, value)
		}
		close(ch)
		set.RUnlock()
	}()
	return ch
}
func main() {
	th := threadSafeSet{
		s: []interface{}{"1", "2"},
	}
	v := <-th.Iter()
	fmt.Println(v)
}
```
# 考点：方法接收者是值还是指针问题

以下代码能编译过去吗？为什么？
```go
package main

import (
	"fmt"
)

type People interface {
	Speak(string) string
}
type Stduent struct{}

func (stu *Stduent) Speak(think string) (talk string) {
	if think == "bitch" {
		talk = "Youare a good boy"
	} else {
		talk = "hi"
	}
	return
}
func main() {
	var peo People = Stduent{}
	think := "bitch"
	fmt.Println(peo.Speak(think))
}

```
输出：
```
./prog.go:21:6: cannot use Stduent literal (type Stduent) as type People in assignment:
	Stduent does not implement People (Speak method has pointer receiver)
```
解答：
> 编译不通过！ 方法接收者是指针时，接口的值只能是指针。

# 考点：interface内部结构

以下代码打印出来什么内容，说出为什么。
```go
package main

import (
	"fmt"
)

type People interface {
	Show()
}
type Student struct{}

func (stu *Student) Show() {
}
func live() People {
	var stu *Student
	return stu
}
func main() {
	if live() == nil {
		fmt.Println("AAA")
	} else {
		fmt.Println("BBB")
	}
}
```
输出：
```
BBB
```
解答：
> 很经典的题！这个考点是很多人忽略的interface内部结构。 go中的接口分为两种：

一种是空接口类似这样：
```
var in interface{}
```
他的底层结构如下：
```go
 //空接口
type eface struct {
    //类型信息
    _type *_type
    
    //指向数据的指针(go语言中特殊的指针类型unsafe.Pointer类似于c语言中的void*)
    data  unsafe.Pointer 
}
```

另一种如题目：
```
type People interface {
    Show()
}
```
他们的底层结构如下：
```
//带有方法的接口
type iface struct {  
    //存储type信息还有结构实现方法的集合
    tab  *itab 
    //指向数据的指针(go语言中特殊的指针类型unsafe.Pointer类似于c语言中的void*)
    data unsafe.Pointer
}
type itab struct {
    inter  *interfacetype //接口类型
    _type  *_type         //结构类型
    link   *itab
    bad    int32
    inhash int32
    fun    [1]uintptr     //可变大小方法集合
}
    
type _type struct {
    size       uintptr //类型大小
    ptrdata    uintptr //前缀持有所有指针的内存大小
    hash       uint32  //数据hash值
    tflag     tflag
    align      uint8   //对齐
    fieldalign uint8   //嵌入结构体时的对齐
    kind       uint8   //kind 有些枚举值kind等于0是无效的
    alg       *typeAlg //函数指针数组，类型实现的所有方法
    gcdata    *byte   str       nameOff
    ptrToThis typeOff
}
```
> 可以看出iface比eface 中间多了一层itab结构。 itab 存储_type信息和[]fun方法集，从上面的结构我们就可得出，因为data指向了nil 并不代表interface 是nil， 所以返回值并不为空，这里的fun(方法集)定义了接口的接收规则，在编译的过程中需要验证是否实现接口

# 考点：type的使用

是否可以编译通过？如果通过，输出什么？
```go
package main


func main() {
	i := GetValue()
	switch i.(type) {
	case int:
		println("int")
	case string:
		println("string")
	case interface{}:
		println("interface")
	default:
		println("unknown")
	}
}
func GetValue() int {
	return 1
}
```
输出：
```
./prog.go:6:2: cannot type switch on non-interface value i (type int)
```

解析：
> 编译失败，因为type只能使用在interface

# 考点：函数返回值命名规则
下面函数有什么问题？
```go
func Mui(x, y int) (sum int, error) {
	return x + y, nil
}
```

解析:
> 在函数有多个返回值时，只要有一个返回值有指定命名，其他的也必须有命名；如果返回值有有多个返回值必须加上括号；如果只有一个返回值并且有命名也需要加上括号。
>
> 此处函数第一个返回值有sum名称，第二个未命名，所以错误。

# 考点: defer、return、函数返回值的执行顺序
14.是否可以编译通过？如果通过，输出什么？

```go
package main

func main() {
	println(DeferFunc1(1))
	println(DeferFunc2(1))
	println(DeferFunc3(1))
}

func DeferFunc1(i int) (t int) {
	t = i
	defer func() {
		t += 3
	}()
	return t
}
func DeferFunc2(i int) int {
	t := i
	defer func() {
		t += 3
	}()
	return t
}
func DeferFunc3(i int) (t int) {
	defer func() {
		t += i
	}()
	return 2
}
```
输出：
```
4
1
3
```
解析：
> 需要明确一点是defer需要在函数结束前执行。 函数返回值名字会在函数起始处被初始化为对应类型的零值并且作用域为整个函数 
>
> defer、return、返回值三者的执行顺序应该是：return最先给返回值赋值；接着defer开始执行一些收尾工作；最后RET指令携带返回值退出函数。
>
> DeferFunc1有函数返回值t作用域为整个函数，在return之前defer会被执行，所以t会被修改，返回4; DeferFunc2函数中t的作用域为函数，返回1;DeferFunc3返回3

# 考点：make和new的区别

是否可以编译通过？如果通过，输出什么？
```go
package main

import "fmt"

func main() {
	list := new([]int)
	list = append(list, 1)
	fmt.Println(list)
}

```
输出：
```
./prog.go:7:15: first argument to append must be slice; have *[]int
```
解析:
> golang中只有三种引用类型它们分别是切片slice、字典map、管道channel。其它的全部是值类型，引用类型可以简单的理解为指针类型，它们都是通过make完成初始化。
>
> 内置函数 new 计算类型大小，为其分配零值内存，返回指针。而 make 会被编译器翻译 成具体的创建函数，由其分配内存和初始化成员结构，返回对象而非指针。
>
> 修改为： list := make([]int,0)

# 考点：结构体比较
是否可以编译通过？如果通过，输出什么？
```go
package main

import "fmt"

func main() {
	sn1 := struct {
		age  int
		name string
	}{age: 11, name: "qq"}
	sn2 := struct {
		age  int
		name string
	}{age: 11, name: "qq"}
	if sn1 == sn2 {
		fmt.Println("sn1== sn2")
	}

	sm1 := struct {
		age int
		m   map[string]string
	}{age: 11, m: map[string]string{"a": "1"}}
	sm2 := struct {
		age int
		m   map[string]string
	}{age: 11, m: map[string]string{"a": "1"}}
	if sm1 == sm2 {
		fmt.Println("sm1== sm2")
	}
}
```
输出：
```
./prog.go:26:9: invalid operation: sm1 == sm2 (struct containing map[string]string cannot be compared)
```
解析:
> 结构体是相同的，但是结构体属性中有不可以比较的类型，如map,slice,function。 所以该结构体不可比较。
>
> 可以使用reflect.DeepEqual进行比较

# 考点：iota

是否可以编译通过？如果通过，输出什么？

```go
package main

import (
	"fmt"
)

const (
	x = iota
	y
	z = "zz"
	k
	p = iota
)

func main() {
	fmt.Println(x, y, z, k, p)
}
```
输出：
```
0 1 zz zz 4
```

解析:
> const声明中,每新增一行常量声明将使iota计数一次;iota仅能在const声明中使用;


# 考点:常量的内存分配

下面函数有什么问题？
```go
package main

const cl = 100

var bl = 123

func main() {
	println(&bl, bl)
	println(&cl, cl)
}
```
输出：
```
cannot take the address of cl
```

解析:
> 常量不同于变量的在运行期分配内存，常量通常会被编译器在预处理阶段直接展开，作为指令数据使用

# 考点：goto 
编译执行下面代码会出现什么?
```go
package main

func main() {
	for i := 0; i < 10; i++ {
	loop:
		println(i)
	}
	goto loop
}
```
输出：
```
./prog.go:8:7: goto loop jumps into block starting at ./prog.go:4:26
```
解析:
> goto不能跳转到其他函数或者内层代码

# 考点：Go 1.9 新特性 Type Alias

25.编译执行下面代码会出现什么?

```go
package main

import "fmt"

type User struct {
}
type MyUser1 User
type MyUser2 = User

func (i MyUser1) m1() {
	fmt.Println("MyUser1.m1")
}
func (i User) m2() {
	fmt.Println("User.m2")
}
func main() {
	var i1 MyUser1
	var i2 MyUser2
	i1.m1()
	i2.m2()
}
```
输出：
```
MyUser1.m1
User.m2
```

解析:
> type MyInt1 int
> type MyInt2 = int
> 第一行代码是基于基本类型int创建了新类型MyInt1(type defintion)，第二行是创建的一个int的类型别名MyInt2(type alias)，注意类型别名的定义是 `=`。

```go
package main

import "fmt"

type MyInt1 int
type MyInt2 = int

func (i MyInt1) m1() {
	fmt.Println("MyInt1.m1")
}

func (i MyInt2) m2() {
	fmt.Println("MyInt2.m2")
}

func main() {
	var i1 MyInt1
	var i2 MyInt2
	i1.m1()
	i2.m2()
}
```
输出：
```
./prog.go:12:6: cannot define new methods on non-local type int
./prog.go:20:4: i2.m2 undefined (type int has no field or method m2)
```
解析：
> 因为int是一个非本地类型，所以我们不能为其增加方法。

# 考点：panic仅有最后一个可以被revover捕获
编译执行下面代码会出现什么?

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println(err)
		} else {
			fmt.Println("fatal")
		}
	}()
	defer func() {
		panic("defer panic")
	}()
	panic("panic")
}

func main1() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("++++")
			f := err.(func() string)
			fmt.Println(err, f(), reflect.TypeOf(err).Kind().String())
		} else {
			fmt.Println("fatal")
		}
	}()
	defer func() {
		panic(func() string {
			return "defer panic"
		})
	}()
	panic("panic")
}
```
输出：
```
main1:
defer panic

main:
++++
0xe03e0 defer panic func
```

解析:
> 触发panic("panic")后顺序执行defer，但是defer中还有一个panic，所以覆盖了之前的panic("panic")
