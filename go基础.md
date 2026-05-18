# Go 和 Java 的区别，优缺点

## 核心区别

| 维度   | Go                  | Java          |
| ---- | ------------------- | ------------- |
| 定位   | 云原生 / 高并发 / 工程效率    | 企业级生态         |
| 编译   | 静态编译成本地二进制          | 编译成 JVM 字节码   |
| 并发模型 | Goroutine + Channel | Thread + Lock |
| GC   | 轻量、低延迟              | 成熟、功能强        |
| 部署   | 单二进制部署              | 依赖 JVM        |
| 泛型   | 较晚支持                | 很成熟           |
| 面向对象 | 弱 OOP，无继承           | 强 OOP         |
| 启动速度 | 快                   | 相对慢           |
| 开发效率 | 简洁                  | 规范化强          |

---

## Go 的优点

### 1. 高并发简单

Go 原生支持协程：

```go
go func() {
    fmt.Println("hello")
}()
```

相比 Java：

```java
new Thread(() -> {
    System.out.println("hello");
}).start();
```

Go 的协程创建成本极低。

---

### 2. 部署方便

Go：

```bash
go build
./app
```

直接得到单个可执行文件。

Java：

```bash
java -jar app.jar
```

需要 JVM 环境。

---

### 3. 编译快

Go 编译速度非常快，适合云原生微服务。

---

### 4. 语言简单

Go 故意减少复杂特性：

* 无继承
* 无注解地狱
* 无复杂泛型
* 无 overloaded

适合团队协作。

---

## Go 的缺点

### 1. 生态不如 Java 完整

Java 在：

* 企业中间件
* 大数据
* ORM
* 企业框架

方面积累更深。

---

### 2. 错误处理比较“丑”

Go：

```go
if err != nil {
    return err
}
```

大量重复。

---

### 3. 泛型能力较弱

Go 泛型设计偏保守。

---

## Java 的优点

* 生态成熟
* Spring 全家桶强大
* JVM 调优成熟
* 面向对象体系完整

---

## Java 的缺点

* 项目偏重
* 启动慢
* 内存占用大
* 高并发编写复杂

---

## 面试高频总结

### 为什么 Go 更适合云原生？

因为：

* 协程轻量
* 部署简单
* 编译快
* 内存占用低
* 高并发友好

---

# GMP 模型

Go 调度器核心：

* G：goroutine
* M：内核线程
* P：调度器 Processor

Go 采用 GMP 调度模型。([Go语言学习][1])

---

# G、M、P 分别是什么

## G（Goroutine）

用户态协程。

特点：

* 极轻量
* 初始栈只有几 KB
* 可以创建几十万级别

---

## M（Machine）

真正的内核线程。

由 OS 调度。

---

## P（Processor）

调度上下文。

作用：

* 管理本地 G 队列
* 绑定 M
* 执行调度

P 的数量默认等于 CPU 核数。([Go语言学习][1])

---

# GMP 工作流程

```mermaid
flowchart LR

G1[Goroutine]
G2[Goroutine]
G3[Goroutine]

P[Processor 本地队列]

M[Machine 内核线程]

CPU[CPU]

G1 --> P
G2 --> P
G3 --> P

P --> M
M --> CPU
```

---

# 本地队列 + 全局队列

每个 P 有自己的本地队列。

优势：

* 减少锁竞争
* 提高并发性能

---

# Work Stealing 偷 G

当某个 P 没任务：

会先从全局队列拿G，如果全局队列没有G，再尝试从其他 P 偷一半 G。

```mermaid
flowchart TD

Local[本地队列为空]

Local --> Global[尝试全局队列]

Global -->|有G| Run1[执行G]

Global -->|没有| Steal[偷其他P的本地队列]

Steal -->|偷到| Run2[执行G]

Steal -->|偷不到| Netpoll[检查网络poller]

Netpoll -->|还没有| Park[线程休眠]
```

这是 Go 高并发的重要设计。

---

# Hand Off

如果 G 阻塞系统调用：

```go
syscall.Read(...)
```

对应 M 会阻塞。

此时：

* P 会解绑 M
* 找新的 M 继续执行

避免整个调度卡死。([深入Go语言之旅][2])

---

# 抢占式调度

早期 Go：

协作式调度。

问题：

死循环会卡住。

Go1.14 后：

支持基于信号的抢占调度。([Go语言学习][1])

---

# Channel

Go 的核心并发通信机制。

Go 推崇：

> 不要通过共享内存通信，而要通过通信共享内存。

**Channel**的底层是一个名为**hchan**的结构体，包含一个**环形缓冲区**、发送和接收的索引指针，以及两个**goroutine等待队列**。

**发送方**和**接收方**通过**互斥锁**保证并发安全，当缓冲区满或空时，goroutine会被挂起到对应的**等待队列**中。

**Channel**在三种情况下会触发**panic**：向已关闭的Channel发送数据、重复关闭Channel、关闭值为**nil**的Channel。

![img](https://file1.kamacoder.com/i/algo/go_channel_principle.jpg)

---

## Channel 底层结构

底层是：

* 环形队列
* 等待队列（双向链表）

结构：

![img](https://img2024.cnblogs.com/blog/1497240/202407/1497240-20240705205844758-747817590.png)

---

### 无缓冲 Channel

```go
ch := make(chan int)
```

特点：

* 必须同时有发送和接收
* 同步通信

```go
ch <- 1 // 阻塞
```

直到有人接收。

---

### 有缓冲 Channel

```go
ch := make(chan int, 3)
```

特点：

* buffer 未满：发送不阻塞
* buffer 未空：接收不阻塞

---

## 阻塞条件

### 发送阻塞

发生条件：

* 无缓冲：没人接收
* 有缓冲：buffer 满

---

### 接收阻塞

发生条件：

* 无缓冲：没人发送
* 有缓冲：buffer 空

---

## close 后特点

```go
close(ch)
```

---

### 继续读

可以继续读取剩余数据：

```go
v, ok := <-ch
```

* ok=true：正常数据
* ok=false：channel 已关闭

---

### 遍历行为

```go
for v := range ch {

}
```

会：

* 读完剩余数据
* 自动退出循环

---

### close 已关闭 channel

会 panic：

```go
panic: close of closed channel
```

---

# Map

---

## 底层结构

* **Go的map底层核心结构是`hmap`，其中关键字段包括桶数组指针`buckets`、元素数量`count`、扩容状态`oldbuckets`等。**

  每个桶对应一个`bmap`结构，包含以下核心设计：

  1. **tophash数组**：存储每个key哈希值的高8位，用于快速比对，避免无效的全量key比较
  2. **键数组和值数组**：以非交叉方式分开存储，即先存所有key再存所有value，减少内存对齐损耗
  3. **overflow指针**：指向溢出桶，当当前桶存满8个元素后，通过链表方式串联新桶

  查找时，Go**先用哈希值低位确定桶编号，再用高8位与tophash比对，命中后才做完整key比较**，大幅提升查询效率。

  扩容分为两种策略：

  1. **翻倍扩容**：负载因子超过6.5时触发，桶数量翻倍
  2. **等量扩容**：溢出桶过多但元素不多时触发，整理内存碎片

  扩容采用渐进式迁移，每次写操作时迁移少量桶，避免长时间阻塞。

  ![img](https://file1.kamacoder.com/i/algo/go_map.jpg)

---

## Map 查找流程

```mermaid
flowchart LR

Key --> Hash
Hash --> Bucket
Bucket --> Value
```

---

## 扩容时机

### 1. 装载因子过高

元素太多。

---

### 2. 溢出桶太多

哈希冲突严重。

---

## 渐进式扩容

Go map 不会一次性搬迁。

而是：

* 每次操作搬一点

避免长时间卡顿。

---

## Map 并发不安全

错误示例：

```go
go func() {
    m["a"] = 1
}()

go func() {
    m["b"] = 2
}()
```

可能：

```text
fatal error: concurrent map writes
```

---

### 解决方案

#### 1. 加读写锁（大多数场景）

```go
type SafeMap struct {
    mu sync.RWMutex
    m map[string]int
}
```

---

#### 2. sync.Map

适合：

* 读多写少
* 用户信息缓存
* 白名单
* 并发场景

大量 Store/Delete下性能很差

---

# Slice

---

## 底层结构

Slice 本质：

```go
type slice struct {
    ptr *array
    len int
    cap int
}
```

---

## len 和 cap

```go
s := make([]int, 3, 5)
```

* len=3
* cap=5

---

## 扩容规则

append 时：

### 小于 256

一般扩容为 2 倍。

---

### 大于 256

逐渐降低增长倍率。

避免浪费内存。

---

## Slice 引用陷阱

```go
arr := [5]int{1,2,3,4,5}

s1 := arr[:2]
s2 := arr[:3]

s1[0] = 100
```

s2 也会变化。

因为：

共享底层数组。

---

## append 陷阱

```go
s1 := []int{1,2,3}
s2 := s1

s1 = append(s1, 4)
```

可能：

* 共用数组
* 也可能扩容后分离

面试很爱问。

----

# new() 和 make() 的区别

new 和 make 的根本区别在于用途与返回值。

**new** 是一个通用内存分配器，会在堆上分配一块足够容纳类型 T 的内存空间，并将这块内存的所有位设置为零，最后返回指向这块内存的指针 *T。

- **对于值类型**（如基本类型、结构体），会得到一个指向有效零值的指针。
- **对于引用类型**（slice, map, channel），返回的是指向 nil 值的指针。例如，new([]int) 返回一个类型为 *[]int 的指针，但其指向的切片本身是 nil，**无法直接使用**。

而**make** 是一个专用的构造函数，仅用于切片、映射和通道这三种内建的引用类型。它会执行复杂初始化并直接返回一个已初始化、立即可用的值 ，而非指针。

- **对于切片**：`make([]T, len, cap)` 会分配一个底层数组（容量为 cap），并创建一个**切片头**（包含指向数组的指针、长度和容量）来管理这块数组。
- **对于映射**：`make(map[K]V)` 会初始化一个哈希表结构，包括创建桶等内部数据结构，使其可以立即接收键值对。
- **对于通道**：`make(chan T, size)` 会创建通道所需的环形缓冲区以及同步用的互斥锁等结构，使其能够进行协程间的通信。

因此，选择使用哪个函数取决于具体场景：

当需要立即可用的切片、映射或通道时，必须使用 make。当需要获取一个指向某类型零值的指针时，则使用 new。

![image](https://file1.kamacoder.com/i/algo/go_newVSmake.jpg)

---

# 接口（interface）

Go语言中接口比较的核心在于其内部结构。每个接口值都由两部分组成：

- **动态类型**：接口所持有的具体值的类型。
- **动态值**：接口所持有的具体值。

只有当这两个部分都完全相同时，两个接口才相等。

两个 interface 相等有以下2种情况：

- 动态类型T相同，且对应的动态值V相等。
- 两个 interface 均等于nil (此时 V 和 T 都处于 unset 状态)。

但如果接口底层封装的是**不可比较的类型**，如切片、映射和函数，那么直接比较这两个接口会导致**运行时panic**。这是因为Go语言本身不允许直接比较这些类型。

![image](https://file1.kamacoder.com/i/algo/go_interface.jpg)

## 面试官可能追问

Q1：谈谈 interface 的底层原理。

A1：Go语言中接口的底层通过两种结构实现：**eface** 和 **iface**，分别对应空接口 interface{} 和带方法的接口。

- **eface** 结构包含两个字段：

  - **_type 指针**，指向接口所持有值的具体类型元信息；
  - **data 指针**，指向实际的数据值。

  这使得空接口可以承载任何类型的值。

- **iface** 则用于带有方法的接口，它除了包含 data 指针外，还有一个关键的 **tab 字段**，指向一个 itab 结构。

  itab 是接口动态调度的核心，它存储了接口类型本身的信息 (inter)、具体值的类型信息 (_type)，以及一个名为 fun 的函数指针数组。

  fun 数组里保存了具体类型所实现的接口方法的地址，从而在通过接口调用方法时，能够找到正确的函数执行。

所以当将一个具体类型的值赋值给接口变量时，Go 运行时会在必要时构建相应的 itab 或设置 _type。进行类型断言时，则会通过比较 itab 中的类型信息或 _type 指针来完成。

Q2：Go语言中interface有哪些应用场景？

A2：在Go语言中，接口的应用场景十分广泛，主要用来实现多态、降低耦合并提升代码的灵活性与可测试性。

1. **实现多态与行为抽象**，允许不同类型对象对同一消息作出响应，编写通用代码；
2. **解耦与依赖注入**，可以降低模块耦合度，提高代码的可测试性和可维护性；
3. **错误处理**，通过 error 接口提供统一、可扩展的错误处理机制。
4. Go标准库也大量使用接口（如 io.Reader 和 io.Writer）来提供**通用的IO**能力，而空接口 interface{} 则常作为容器处理未知类型数据，在JSON解析等场景中使用。

# Defer

---

## 后进先出

```go
defer fmt.Println(1)
defer fmt.Println(2)
```

输出：

```text
2
1
```

类似栈。

---

## return 执行顺序

核心：

```text
return 先赋值
defer 后执行
最后真正返回
```

---

## 经典题

```go
func test() int {
    i := 0

    defer func() {
        i++
    }()

    return i
}
```

返回：

```text
0
```

因为 return 时已经复制返回值。

---

## 命名返回值

```go
func test() (i int) {
    defer func() {
        i++
    }()

    return i
}
```

返回：

```text
1
```

---

## 闭包变量捕获

```go
for i := 0; i < 3; i++ {
    defer func() {
        fmt.Println(i)
    }()
}
```

输出：

```text
3
3
3
```

因为闭包捕获的是变量本身。

---

# 逃逸分析

Go 编译器决定：

变量放：

* 栈
* 堆

---

## 栈的特点

优点：

* 分配快
* 自动回收

---

## 堆的特点

缺点：

* GC 压力大

---

## 哪些会逃逸

---

### 1. 返回局部变量指针

```go
func test() *int {
    a := 10
    return &a
}
```

a 必须放堆上。

---

### 2. 被 interface 引用

```go
fmt.Println(a)
```

可能逃逸。

---

### 3. 大对象

栈放不下。

---

## 查看逃逸分析

```bash
go build -gcflags="-m"
```

---

# Mutex / RWMutex

---

## Mutex

互斥锁：

```go
var mu sync.Mutex
```

同一时间：

只能一个协程进入。

---

## RWMutex

读写锁：

```go
var mu sync.RWMutex
```

支持：

* 多读
* 单写

---

## 使用场景

### Mutex

适合：

* 写多
* 简单场景

---

### RWMutex

适合：

* 读多写少

---

## 注意点

RWMutex：

* 写锁优先级高
* 不可锁升级

错误：

```go
RLock()
Lock()
```

会死锁。

---

# GC 三色标记

注意 goroutine 死亡后栈也会被回收，不需要 GC 的参与。只有那些需要长期存在的对象才会被分配到需要进行垃圾回收的堆中。

Go GC 核心：

**三色标记法**。

---

## 三种颜色

| 颜色 |                        含义                         |
| :--: | :-------------------------------------------------: |
| 白色 |           未被扫描，默认是垃圾，等待回收            |
| 灰色 |      自身已被扫描，但**引用的子对象还没扫描**       |
| 黑色 | 自身 + 所有引用对象都扫描完毕，**确定存活，不回收** |

---

## 变色流程

```mermaid
stateDiagram-v2
    白色对象 --> 灰色对象 : 从根对象遍历可达
    灰色对象 --> 黑色对象 : 递归扫描完所有引用
    白色对象 --> 回收 : 全程不可达
```

---

## 核心思想

从根对象开始遍历：

* 栈
* 全局变量
* 寄存器

找到所有活对象。

---

## STW

Stop The World：

GC 时暂停程序。

---

## 为什么要降低 STW

否则延迟会非常高。

---

## 写屏障

GC 时：

用户代码可能修改引用关系。

例如：

```
a.ptr = b // b 是白色，a是黑色
```

这种写操作如果不记录：

```
可能导致黑对象指向白对象
```

从而：

```
白对象漏标记 → 被误回收
```

所以必须开启写屏障。

写屏障用于：

维护三色不变性。

降低 STW 时间。

----

## 四大执行阶段

1. 最开始，所有对象都是白色：![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7d949b4652b4224a4db0b845ca44957~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1634&h=964&s=169352&e=png&b=fdfdfd)

2. **标记准备 MarkInit**

触发**短暂 STW**；初始化 GC 环境、扫描所有根对象，把可达对象标为灰色，并开启写屏障

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29241fde7049405fa20176b082015dbe~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1582&h=932&s=189221&e=png&b=fdfdfd)

3. **并发标记 Mark**

**不阻塞业务协程**；GC 协程和用户协程并发运行，

从灰色对象递归遍历引用，灰色→黑色；全程保持写屏障并正常执行业务代码。（注意这个阶段只有根节点会变黑）

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9521dafb9ad4134b6ddfeb90462ffc7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1586&h=916&s=191283&e=png&b=fdfdfd)

4. **标记终止 MarkTerm**

再次**短暂 STW**，关闭写屏障；因为并发期间用户协程可能修改对象引用，需要重新扫描栈、补全标记，消除漏标。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d75e2b4bc1594f558bedd4d6727139f3~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1598&h=916&s=164400&e=png&b=fdfdfd)

5. **并发清除 Sweep**

再次并行；遍历堆内存，把**残留白色对象**统一回收，内存归还堆，不阻塞业务。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb3b5d034b024d3f9e853a1b0a458473~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1578&h=908&s=150451&e=png&b=fdfdfd)

> 重点：Go GC **只有两次短 STW**，全程大部分时间并发，卡顿极低。

## 写屏障（必须讲，面试高频）

### 为什么需要写屏障？

并发标记阶段，**用户协程修改对象引用**会出现两种问题：

1. 黑色对象 引用 白色对象 → 白色对象被漏标，误回收（严重 bug）
2. 产生浮动垃圾：已经标记完的对象变成垃圾，本轮回收不掉

**解决方案：写屏障**

在指针赋值操作前后插入一段逻辑，**强制维护三色标记的约束**：

- 保证黑色对象不会直接引用白色对象
- Go 1.19 后使用 **混合写屏障**，省去栈重扫描，进一步缩短 STW

------

## 五、STW 什么时候发生？

只在两个阶段停顿：

1. 标记准备阶段
2. 标记终止阶段

其余全部并发执行，这是 Go GC 低延迟的核心。

------

## 六、GC 触发时机

1. **内存阈值触发**：堆内存分配量达到增长阈值，自动触发
2. **手动触发**：调用 `runtime.GC()`
3. **定时触发**：后台强制定时 GC，防止长期不回收内存溢出

----

# struct tag

Go 语言的 tag 是**结构体字段的元数据**，以key:"value"形式存在，用分号分隔，通过**反射**获取。

它的主要作用是为结构体字段提供额外信息，影响字段的处理行为。

- 在 JSON 处理中，tag 可指定字段的 JSON 名称、是否忽略空值、是否为 omitempty 等；
- 在 XML 处理中，可指定 XML 元素名、属性等。
- 在 ORM 框架中，tag 用于映射结构体字段到数据库表的列名，如gorm:"column:user_name;index"，指定字段对应的数据库列名和索引属性。
- tag 还可用于表单验证，如validate:"required,email"，为字段添加验证规则。

tag 的使用使得结构体字段具有更强的表达能力，通过反射机制实现了元数据驱动的编程模式，是 Go 语言中处理复杂数据结构的重要工具。

![image](https://file1.kamacoder.com/i/algo/go_tag.jpg)

----

# panic 和 recover

## 基本概念

1. **panic** 的作用是使当前协程的执行被立即中断

   它会停止当前函数的执行，开始回溯调用栈，并依次执行所有已注册的 defer 语句。

   如果没有任何 recover 捕获，程序最终会崩溃并打印堆栈跟踪信息。

   主要使用场景有文件未找到、网络超时等。

2. **recover** 的作用是用于**捕获同一个协程中发生的 panic，阻止其继续向上传播导致程序崩溃**，它只能在 defer 函数中生效。

   主要使用场景有程序启动依赖失败、协程内部致命错误、防止因第三方库的 panic 导致整个服务崩溃等。

![image](https://file1.kamacoder.com/i/algo/go_panicVSrecover.jpg)

----

## 使用场景

- **Web 框架/网关的全局中间件**： 这是最经典的应用场景。比如在使用 Gin 开发服务时，框架的最外层通常会有一个 Recovery 中间件，利用 defer + recover 兜底，捕获异常，记录日志，并给用户返回一个优雅的 500 HTTP 状态码。
- **守护并发协程**： 在后台执行海量异步任务时，通常会封装一个 SafeGo 函数。在开启新协程时，注入 defer recover()，保证单个后台任务的崩溃不会牵连整个主进程。

---

# 反射基础

Go 反射核心包：

```go
reflect
```

---

## 反射作用

运行时：

* 获取类型
* 获取字段
* 动态调用

---

## Type 和 Value

```go
t := reflect.TypeOf(x)
v := reflect.ValueOf(x)
```

---

## 修改值必须传指针

```go
func test(x interface{}) {
    v := reflect.ValueOf(x)

    v.Elem().SetInt(100)
}
```

必须：

```go
test(&a)
```

---

## 常见用途

---

### 1. ORM

GORM：

通过反射解析结构体。

---

### 2. JSON

```go
json.Marshal()
```

底层大量使用反射。

---

### 3. 框架

Gin 参数绑定：

```go
ShouldBind()
```

---

## 反射缺点

---

### 1. 性能差

因为：

* 动态类型检查
* 无法编译期优化

---

### 2. 不安全

容易 panic。

---

# 面试高频总结

## Go 为什么高并发强？

核心：

* GMP 调度
* Goroutine 轻量
* Channel CSP 模型
* 用户态调度

---

## Map 为什么并发不安全？

因为扩容和写入会修改内部结构。

---

## Slice 为什么会“互相影响”？

因为共享底层数组。

---

## defer 为什么容易出错？

因为：

* return/defer 执行顺序
* 闭包变量捕获

是高频陷阱。

---

Go GMP 调度相关内容参考：
([Go语言学习][1])

[1]: https://go.codetrue.net/concurrent-programming/goroutine-scheduling?utm_source=chatgpt.com "GMP 模型原理 | Go语言学习"
[2]: https://go.cyub.vip/gmp/gmp-model/?utm_source=chatgpt.com "调度机制概述 | 深入Go语言之旅"
