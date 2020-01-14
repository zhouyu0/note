###  Golang面试题

[参考资料1](https://yushuangqi.com/blog/2017/golang-mian-shi-ti-da-an-yujie-xi.html)

[参考资料2](https://blog.csdn.net/itcastcpp/article/details/80462619)

**1、写出下面代码输出内容。**

```go
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() { fmt.Println("打印前") }()
	defer func() { fmt.Println("打印中") }()
	defer func() { fmt.Println("打印后") }()

	panic("触发异常")
}
```

**2、 以下代码有什么问题，说明原因**

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
// range 之后赋值给stu，地址值一样，最后都指向了最后一个元素
```

**3、下面代码会输出什么，说明理由**

```go
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("i: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("i: ", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

**4、下面代码会输出什么**

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
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```

**5、下面代码会触发异常吗？请详细说明**

```go
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

答： 有可能触发异常，是随机事件。

单个chan如果无缓冲时，将会阻塞。但结合 select可以在多个chan间等待执行。有三点原则：

1. select 中只要有一个case能return，则立刻执行。
2. 当如果同一时间有多个case均能return则伪随机方式抽取任意一个执行。
3. 如果没有一个case能return则可以执行”default”块。

此考题中的两个case中的两个chan均能return，则会随机执行某个case块。故在执行程序时，有可能执行第二个case，触发异常

**6、下面会输出什么？**

```go
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
	a := 1
	b := 2
	defer calc("1", a, calc("10", a, b))
	a = 0
	defer calc("2", a, calc("20", a, b))
	b = 1
}
```

在解题前需要明确两个概念：

* defer是在函数末尾的return前执行，先进后执行，具体见问题1。
* 函数调用时 int 参数发生值拷贝。

不管代码顺序如何，defer calc func中参数b必须先计算，故会在运行到第三行时，执行calc("10",a,b)输出：10 1 2 3得到值3，将cal("1",1,3)存放到延后执执行函数队列中。

执行到第五行时，现行计算calc("20", a, b)即calc("20", 0, 2)输出：20 0 2 2得到值2,将cal("2",0,2)存放到延后执行函数队列中。

执行到末尾行，按队列先进后出原则依次执行：cal("2",0,2)、cal("1",1,3) ，依次输出：2 0 2 2、1 1 3 4 。

**7、写出下列输出内容**

```go
func main() {
	s := make([]int, 5)
	s = append(s, 1, 2, 3)
	fmt.Println(s)
}
```

0 0 0 0 0 1 2 3

当执行make([]int,5)时返回的是一个含义默认值(int的默认值为0)的数组:[0,0,0,0,0]。而append函数是便是在一个数组或slice后面追加新的元素，并返回一个新的数组或slice。

**8、下面代码有什么问题**

```go
type UserAges struct {
	ages map[string]int
	sync.Mutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}
```

答： 在执行 Get方法时可能被panic

解析: 虽然有使用sync.Mutex做写锁，但是map是并发读写不安全的。map属于引用类型，并发读写时多个协程见是通过指针访问同一个地址，即访问共享变量，此时同时读写资源存在竞争关系。会报错误信息:“fatal error: concurrent map read and map write”。

可以在在线运行中执行，复现该问题。那么如何改善呢? 当然Go1.9新版本中将提供并发安全的map。首先需要了解两种锁的不同：

1. `sync.Mutex`互斥锁
2. `sync.RWMutex`读写锁，基于互斥锁的实现，可以加多个读锁或者一个写锁。

**9、下面迭代会有什么问题**

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

内部迭代出现阻塞。默认初始化时无缓冲区，需要等待接收者读取后才能继续写入。

解析: chan在使用make初始化时可附带一个可选参数来设置缓冲区。默认无缓冲，题目中便初始化的是无缓冲区的chan，这样只有写入的元素直到被读取后才能继续写入，不然就一直阻塞。

设置缓冲区大小后，写入数据时可连续写入到缓冲区中，直到缓冲区被占满。从chan中接收一次便可从缓冲区中释放一次。可以理解为chan是可以设置吞吐量的处理池。

**10、下面代码能编译通过吗**

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
		talk = "You are a good boy"
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

编译失败，值类型 Student{} 未实现接口People的方法，不能定义为 People类型。

解析: 考题中的 `func (stu *Stduent) Speak(think string) (talk string)` 是表示结构类型`*Student`的指针有提供该方法，但该方法并不属于结构类型`Student`的方法。因为struct是值类型。

**11、golang 调度器模型**

**12、select是阻塞还是非阻塞**

**13、make和new的区别**

14、是否能编译通过？为什么

```go
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

编译失败，因为type只能使用在interface

15、输出什么

```go
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

需要明确一点是defer需要在函数结束前执行。 函数返回值名字会在函数起始处被初始化为对应类型的零值并且作用域为整个函数 DeferFunc1有函数返回值t作用域为整个函数，在return之前defer会被执行，所以t会被修改，返回4; DeferFunc2函数中t的作用域为函数，返回1;DeferFunc3返回3
