# Golang 学习笔记

## 目录

- [基础数据结构](#基础数据结构)
  - [数组](#数组)
  - [切片](#切片)
  - [map](#map)
- [并发与通信](#并发与通信)
  - [channel](#channel)
- [类型系统与抽象](#类型系统与抽象)
  - [接口](#接口)
- [并发控制与同步](#并发控制与同步)
  - [context](#context)
  - [计时器](#计时器)
  - [sync包](#sync包)
- [语言特性与工具](#语言特性与工具)
  - [defer](#defer)
  - [select](#select)
  - [unsafe包](#unsafe包)
  - [泛型](#泛型)
  - [反射](#反射)
- [运行时与内存](#运行时与内存)
  - [并发调度](#并发调度)
  - [内存管理](#内存管理)
  - [垃圾回收](#垃圾回收)
- [网络编程](#网络编程)
  - [Go网络编程](#go网络编程)

## 基础数据结构

### 数组

#### 核心概念

- 数组长度在初始化后不可变。
- 元素类型相同但长度不同的数组，在 Go 中是不同类型。
- 编译期数组类型由元素类型 `Elem` 和长度 `Bound` 共同决定。

**速记**

- `[3]int` 和 `[4]int` 不是同一种类型。
- 数组更偏“固定大小值类型”，切片更偏“可增长视图”。
#### 初始化方式

| 方式 | 示例 | 说明 |
| --- | --- | --- |
| 显式指定长度 | `[3]int{1,2,3}` | 长度直接写死 |
| 使用 `...` 推导长度 | `[...]int{1,2,3}` | 编译器自动推导元素个数 |

- `[...]T` 本质是语法糖，编译期仍会转成固定长度数组。

#### 编译器优化
编译器初始化数组字面量时，常见有两种处理策略：

- 元素数量小于等于 `4`：
  直接在栈上初始化。
- 元素数量大于 `4`：
  先放到静态存储区，再整体赋值给目标数组。

```
var arr [3]int
arr[0] = 1
arr[1] = 2
arr[2] = 3
```

当元素数量大于 4 个时，会将数组中的元素放置到静态存储区初始化然后将该临时变量赋值给当前的数组；

```
var arr [5]int
statictmp_0[0] = 1
statictmp_0[1] = 2
statictmp_0[2] = 3
statictmp_0[3] = 4
statictmp_0[4] = 5
arr = statictmp_0
```

#### 越界检查

- 数组访问和赋值最终会转成直接内存读写。
- 常量下标越界：很多情况可在编译期发现。
- 变量下标越界：通常需要运行时检查。
- 运行时发现数组、切片、字符串越界时会直接 `panic`。

**速记**

- 常量越界：偏编译期报错。
- 变量越界：偏运行时 `panic`。

### 切片

#### 核心概念

- 切片是更常用的数据结构，本质上是对底层数组的一层抽象。
- **切片长度可变，容量不足时会触发扩容。**
- **多个切片可能共享同一个底层数组。**
- 切片的很多行为依赖运行时完成。

| 字段 | 含义 |
| --- | --- |
| `Data` | 底层数组地址 |
| `Len` | 当前长度 |
| `Cap` | 当前容量 |

```
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

#### 创建方式
- 常见创建方式：
  - `var s []int`：得到 `nil slice`
  - `[]int{1,2,3}`：字面量创建
  - `make([]int, len, cap)`：显式指定长度和容量
  - `a[low:high:max]`：从数组或切片截取
- 新旧切片是否相互影响，关键看是否共享底层数组。
- **一旦 `append` 触发扩容，新切片可能指向新数组，和旧切片脱钩。**
- `high` 和 `max` 不能超过原切片或原数组的 `cap`。
- `nil slice` 和空切片都可以继续 `append`。

| 形式 | 长度 | 容量 | 与 `nil` 比较 |
| --- | --- | --- | --- |
| `var s []int` | `0` | `0` | `true` |
| `[]int{}` | `0` | `0` | `false` |
| `make([]int, 0)` | `0` | `0` | `false` |

#### 编译期优化

- `len()`、`cap()` 常可在编译期直接替换。
- 下标访问最终会转成直接地址访问。
- `range` 遍历会被编译器展开成更简单的循环形式。

#### 追加与扩容
- `append` 会先判断追加后长度是否超过容量。
- 如果容量足够：直接写入底层数组。
- 如果容量不足：**申请新数组、拷贝旧数据、返回新切片。**
- 可以一次追加多个值，也可以通过 `...` 追加另一个切片。

**常见扩容规则**

- 目标容量小于当前容量两倍时：
  - 当前容量 `< 1024`：通常翻倍
  - 当前容量 `>= 1024`：通常增长约 `25%`
- 目标容量大于当前容量两倍时：
  - 直接使用目标容量
- 最终还会做内存对齐

**速记**

- 判断是否共享底层数组，最关键看 `append` 后有没有扩容。
- 扩容的代价不只是申请内存，还包括整块数据复制。

#### 拷贝
- `copy()` 会通过 `memmove` 做整块内存复制。
- 相比逐元素拷贝，性能通常更好。

#### 使用注意
注意:

- 大切片扩容或复制时，可能发生大规模内存拷贝。
- 切片作为函数参数传递时，**传递的仍然是值，只不过这个值里带有底层数组指针**。
- 修改 `slice[i]` 会影响共享同一底层数组的其他切片。
- 如果函数内部发生扩容，调用方和被调方后续可能不再共享底层数组。
- 如果希望函数能同步修改切片本身（如长度、容量、底层数组），更适合传 `*[]T`。
- **`range` 中拿到的元素是副本，不能直接改原切片元素。**

```go
for _, i := range s {
    i++
}
```

- 上面代码不会修改 `s`，要修改元素必须用索引。

**速记**

- Go 只有值传递。
- “传切片”不是引用传递，“传指针”本质上也是值传递，只是值里存的是地址。

### map

#### 核心概念

- 常见 map 实现思路有两类：
  - 哈希表
  - 搜索树
- 哈希表核心目标：哈希结果尽量均匀。
- 哈希冲突常见处理方式：
  - 开放寻址法
  - 链表法
  - 再哈希法
- Go 采用的是哈希表，并通过桶和溢出桶处理冲突，本质上更接近链地址法。

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| 开放寻址 | CPU 缓存友好、结构简单 | 冲突代价高、删除复杂 |
| 链地址/溢出桶 | 插入删除灵活、装载因子容忍高 | 指针更多、局部性略差 |

**速记**

- Go 的 map 不是红黑树一类结构，核心是**哈希桶**。
- **一个 bucket 最多放 `8` 个 key-value。**

#### 底层结构
```
type hmap struct {
    count     int  //表示当前哈希表中的元素数量；
    flags     uint8 
    B         uint8  /当前哈希表持有的 buckets 数量，但是因为哈希表中桶的数量都 2 的倍数，所以该字段会存储对数，也就是 len(buckets) == 2^B；
    noverflow uint16
    hash0     uint32 //哈希的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入；
    buckets    unsafe.Pointer //指向桶的指针
    oldbuckets unsafe.Pointer //是哈希在扩容时用于保存之前 buckets 的字段，它的大小是当前 buckets 的一半
    nevacuate  uintptr
    extra *mapextra
}

// 桶
type bmap struct {
    tophash [bucketCnt]uint8
}
```



运行时变成

```
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr //当 map 的 key 和 value 都不是指针，并且 size 都小于 128 字节的情况下，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap
    			     //但overflow 的字段，是指针类型的，破坏了 bmap 不含指针的设想，这时会把 overflow 移动到hmap的 extra 字段
}
```


将所有的 key，value 分别绑定到一起，这种形式 key/key/.../value/value/...，
每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来(这就是链表法)

#### 初始化与查找
- map 初始化后，底层是一个 `*hmap`。
- map 作为函数参数时，函数内部对 map 的读写会影响原 map。
- map 使用非加密型哈希做查找，程序启动时会结合 CPU 能力选择实现。

**查找流程**

- 先算哈希值。
- 用低位决定落到哪个 bucket。
- 再用高 8 位快速过滤 bucket 内候选 key。
- 如果当前 bucket 找不到，就继续找 overflow bucket。

**读取行为**

| 写法 | 返回值 |
| --- | --- |
| `v := m[k]` | key 不存在时返回 value 类型零值 |
| `v, ok := m[k]` | `ok` 表示 key 是否存在 |

#### 扩容机制
装载因子 = 元素个数 / bucket 数量。

**触发扩容的两种典型情况**

| 触发条件 | 行为 | 结果 |
| --- | --- | --- |
| 装载因子过高，阈值约 `6.5` | 2 倍扩容 | bucket 数量翻倍 |
| overflow bucket 过多 | 等量扩容 | 重排数据，减少溢出桶 |

**搬迁特点**

- Go 的 map 不会一次性搬完全部数据。
- 扩容后先准备新桶，再在后续写入、删除、访问过程中渐进搬迁。
- 每次最多搬迁少量 bucket，用来降低单次开销。

**遍历特点**

- map 遍历顺序不稳定。
- 多次遍历同一个 map，顺序也可能不同。
- 原因不只是哈希冲突，还包括随机起始 bucket 和渐进扩容。

**删除与内存**

- 删除元素会清零 key/value，并标记对应槽位为空。
- map 扩大后一般不会自动缩回去。

**速记**

- map 无序。
- map 会渐进扩容。
- map 扩大后通常不会缩容。

#### 并发与 key 类型
并发:

- **原生 map 不是并发安全的。**

  map 内部结构（bucket、hash、扩容）会被修改
  并发访问会破坏结构一致性

- 同时读写同一个 map 属于未定义行为，运行时可能直接 `panic`。

- 典型解决方式：
  - 用 `sync.RWMutex` 包住原生 map
  - 使用 `sync.Map`

key 的类型要求是“可比较”。

| 可作为 key | 不可作为 key |
| --- | --- |
| 整数、浮点数、字符串、布尔、指针、channel、接口、数组、结构体、复数 | slice、map、function |

#### 字符串补充
字符串

- 字符串底层可看作“只读字节数组”。
- 运行时结构里主要保存数据地址和长度。
- 因为只读，所以字符串适合在并发场景下安全共享。

```
type StringHeader struct {
    Data uintptr
    Len  int
}
```



- **字符串“修改”本质上通常是创建新字符串，而不是原地改写。**

| 声明方式 | 特点 |
| --- | --- |
| 双引号 | 单行字符串，需要转义 |
| 反引号 | 原始字符串，适合多行内容和 JSON 文本 |

json := `{"author": "draven", "tags": ["golang"]}`

- 单引号声明的是单个字符，类型是 `rune`（本质上是 `int32`）。
- `len([]rune(s))` 统计的是字符数，不是字节数。
- Go 默认字符串编码通常按 UTF-8 处理。

**`string` 与 `[]byte`**

- 标准转换通常会发生内存拷贝。b := []byte(s) 会 **拷贝一份数据**，修改 `b` 不影响 `s`
- 使用 `unsafe.Pointer` 强转可以减少拷贝，但风险高。如果把字符串强转成 `[]byte` 后再修改，可能因为底层仍是只读内存而崩溃。

**速记**

- `len(s)` 是字节数，不一定是字符数。
- 字符串不可变。
- `unsafe` 强转能省拷贝，但会带来安全风险。

## 并发与通信

### channel

#### 核心概念

- Go 倾向使用 CSP 模型做并发协作。
- Goroutine 是执行实体，channel 是通信通道。
- 不带缓冲的 channel 更接近同步通信，带缓冲的 channel 更接近异步通信。

| 类型 | 特点 |
| --- | --- |
| 无缓冲 channel | 发送和接收通常要同时就绪 |
| 有缓冲 channel | 缓冲区未满可先发，缓冲区非空可先收 |

#### 常见 panic 场景
对 channel 的高危操作主要有：

- 向已关闭的 channel 写入
- 关闭 `nil channel`
- 重复关闭同一个 channel

```
close(chan) // 关闭一个通道后,再从通道读会读出类型的零值和标识false
e, ok := <-chan // e==0, ok==false
```

如果需要跨进程通信，建议使用分布式系统的方法来解决。

Channel 分为两种：带缓冲、不带缓冲。对不带缓冲的 channel 进行的操作实际上可以看作“同步模式”，带缓冲的则称为“异步模式”。

#### 底层结构
```go
type hchan struct {
    qcount   uint //当前 Channel 中的元素个数
    dataqsiz uint //Channel 中的循环队列的长度
    buf      unsafe.Pointer //环形队列作为其缓冲区，循环数组sendx 的值等于 dataqsiz 
                              的大小时就会重新回到数组开始的位置。
    elemtype *_type //当前 Channel 能够收发的元素类型
    elemsize uint16 //当前 Channel 能够收发的元素大小
    closed   uint32
    sendx    uint //当前 Channel 发送到了数组中的哪个位置 
    recvx    uint //当前 Channel 接收到了数组中的哪个位置
    recvq    waitq //存储当前 Channel 由于缓冲区空间不足而阻塞的接收Goroutine链表，元素sudog表示一个在等待列表中的 
                     Goroutine
    sendq    waitq //存储当前 Channel 由于缓冲区空间不足而阻塞的发送Goroutine链表
    lock mutex //锁，一个channel同时仅允许被一个goroutine读写
}

ch := make(chan int, 1) 
ch <- 1
<-ch
close(ch)
```
#### 创建
创建时会根据：

- 元素类型
- 缓冲区大小

来为 `hchan` 和底层缓冲区分配内存。

- 无缓冲 channel：只有 `hchan` 本身，没有缓冲数组。
- 有缓冲 channel：还会额外分配底层循环队列。

#### 发送流程
发送时的典型路径：

1. 先加锁。
2. 如果 channel 已关闭，直接 `panic`。
3. 然后分三种情况：

- 情况 A：已有等待接收的 Goroutine
  - 直接把数据拷贝给接收方
  - 唤醒对应 Goroutine
  - 出队并释放锁
- 情况 B：没有等待接收者，但缓冲区还有空间
  - 把数据写入缓冲区
  - 更新计数和索引
  - 释放锁
- 情况 C：没有等待接收者，且无缓冲或缓冲区已满
  - 当前发送方进入阻塞
  - 加入发送等待队列

#### 接收流程

接收时的典型路径：

- 情况 A：有等待发送的 Goroutine，且无缓冲或缓冲区为空
  - 直接从发送方拷贝数据到接收方
- 情况 B：缓冲区里有数据
  - 先从缓冲区取数据
  - 如有等待发送者，再把发送者的数据补进缓冲区
- 情况 C：既没有等待发送者，缓冲区也为空
  - 当前接收方进入阻塞
  - 加入接收等待队列

**接收补充**

- 接收时优先消费缓冲区中的数据。
- 通道发送和接收，本质上都是值拷贝。

| 写法 | 含义 |
| --- | --- |
| `v := <-ch` | 只能拿到值，无法区分“零值”还是“关闭返回” |
| `v, ok := <-ch` | `ok=false` 表示 channel 已关闭且无有效值 |

#### 关闭与单向 channel
关闭：

- 重复关闭或关闭未初始化 channel 都会 `panic`。
- 关闭后：
  - 等待中的接收者会收到零值
  - 等待中的发送者会直接 `panic`
- 如果缓冲区还有数据，关闭后仍然可以继续读出有效值。
- **在不确定是否还有发送者/接收者时，不应贸然关闭 channel。**

单向 channel 只是类型层面的限制，不是不同的底层实现。

| 类型 | 用途 |
| --- | --- |
| `chan T` | 可读可写 |
| `chan<- T` | 只写 |
| `<-chan T` | 只读 |

#### 使用风险
资源泄漏

- channel 可能引发 **goroutine 泄漏**。
- 根因通常是 goroutine 永久阻塞在发送或接收上，而通道状态再也不会变化。

**使用痛点**

- 不改变 channel 状态时，无法直接判断它是否已关闭。
- 关闭已关闭的 channel 会 `panic`。
- 向已关闭的 channel 发送也会 `panic`。

#### 优雅关闭
如何优雅地关闭 channel：

- 使用 `sync.Once` 保证只关闭一次。
- 必要时可用 `defer + recover` 兜底，但这更像防御手段，不是首选设计。

**经验原则**

- 不要从 receiver 侧关闭 channel。
- 多 sender 场景下，不要直接关闭数据 channel。

| 场景 | 推荐做法 |
| --- | --- |
| 1 sender + 1 receiver | sender 关闭数据 channel |
| 1 sender + M receiver | sender 关闭数据 channel |
| N sender + 1 receiver | 用额外信号 channel 或 cancel context 通知停止发送，用 `WaitGroup` 等待所有发送方退出，由一个**协调者**统一关闭 `dataCh` |
| N sender + M receiver | 用额外信号 channel 或 cancel context 控制退出 |

**速记**

- 谁发送，谁更适合负责关闭。
- 多发送方场景，优先关闭“停止信号”，而不是直接关闭数据通道。


## 类型系统与抽象

### 接口

#### 核心概念

- 接口是一组方法签名的集合。
- 接口的作用是解耦调用方和具体实现。
- Go 使用隐式实现：类型只要实现了全部方法，就算实现接口。
- 编译器通常在参数传递、返回值、赋值等场景检查接口实现关系。

| 语言 | 接口实现方式 |
| --- | --- |
| Go | 隐式实现 |
| Java | 显式 `implements` |

```
public class MyInterfaceImpl implements MyInterface {
    public void sayHello() {
        System.out.println(MyInterface.hello);
    }
}
```

#### 方法接收者与实现关系

使用指针接收者的常见理由：

- 需要修改接收者
- 避免复制大对象

**实现关系速查**

| 方法接收者 | 用值赋给接口 | 用指针赋给接口 |
| --- | --- | --- |
| 值接收者 | 可以 | 可以 |
| 指针接收者 | 不可以 | 可以 |

**速记**

- 值接收者的方法集更宽。
- 指针接收者更适合需要修改状态或对象较大的场景。

例如当
type (c *Cat) Quack {}  // 使用结构体指针实现接口
var d Duck = Cat{}      // 使用结构体初始化变量
编译无法通过

#### 底层表示
接口分为两类
我们使用 **iface 结构体表示包含方法的接口**；使用 **eface 结构体表示不包含任何方法的 interface{} 类型**

iface包含接口静态类型,赋值的变量的动态类型和动态值
eface只包含赋值的变量的动态类型和动态值

eface 结构体在 Go 语言的定义是这样的：

```
type eface struct { // 16 bytes
    _type *_type         //指向类型的指针,_type 字段直接复制源类型的
    data  unsafe.Pointer //指向底层数据的指针,赋值时调用 mallocgc 获得一块新内存，把值复制进去，data 再指向这块新内存
}

type _type struct {
    size       uintptr //存储了类型占用的内存空间，为内存空间的分配提供信息
    hash       uint32  //类型的hash值,帮助我们快速确定类型是否相等
    kind       uint8  //类型的编号，有bool, slice, struct 等
    equal      func(unsafe.Pointer, unsafe.Pointer) bool
}
```

Go 语言中的任意类型都可以转换成 interface{} 类型

若某结构体实现了接口A类型,同时也实现了接口B类型
那边将该结构体赋值给接口A后, 进行类型断言 接口A.(接口B),可以得到接口B,
**此时接口A和接口B只有静态类型不同,动态类型和动态值相同**,但接口A和接口B只能使用其定义的方法

#### 接口比较
接口比较:

- 接口比较会同时比较动态类型和动态值。
- 只有动态类型和动态值都为 `nil`，接口本身才等于 `nil`。

```
type iface struct { // 16 bytes
    tab  *itab  //表示接口的类型以及赋给这个接口的实体类型,编译器在编译阶段预先生成好的
    data unsafe.Pointer //指向原始数据的指针,调用 mallocgc 获得一块新内存，把值复制进去，data 再指向这块新内存。
}
```

tab 是接口表指针，指向类型信息；
**data 是数据指针，则指向具体的数据。它们分别被称为动态类型和动态值**
当仅且当这两部分的值都为 nil 的情况下，这个接口值就才会被认为 接口值 == nil

    type itab struct { // 32 bytes
        inter *interfacetype //接口类型,静态类型
        _type *_type  //具体类型,动态类型
    hash  uint32 //，当我们想将 interface类型转换成具体类型时，可以使用该字段快速判断目标类型和具体类型 _type 是否一致
    
    fun   [1]uintptr //接口方法对应的具体数据类型的方法地址.用于动态派发,一般在每次给接口赋值发生转换时会更新此表,这里数组大小为1,存储的是第一个方法的函数指针,更多的方法， 在它之后的内存空间里继续存储,
    }

#### 动态派发

动态派发:

- 接口背后的动态类型可以变化。
- 通过接口调用方法时，运行时会根据动态类型决定调用哪个实现。
- 这就是 Go 中接口支持多态的核心。

#### 类型断言
类型断言

| 写法 | 行为 |
| --- | --- |
| `v, ok := x.(T)` | 安全断言，失败不 panic |
| `v := x.(T)` | 失败会 panic |

断言其实还有另一种形式，就是用在利用 switch 语句判断接口的类型

    switch v := v.(type) {
        case nil:
            fmt.Printf("nil type[%T] %v\n", v, v)
    case Student:
        fmt.Printf("Student type[%T] %v\n", v, v)
    
    case Teacher:
        fmt.Printf("Student type[%T] %v\n", v, v)
    
    case *Student:
        fmt.Printf("*Student type[%T] %v\n", v, v)
    
    default:
        fmt.Printf("unknow\n")
    }

fmt.Println 函数的参数是 interface。对于内置类型，函数内部会用穷举法，得出它的真实类型，然后转换为字符串打印。

而对于自定义类型，首先确定该类型是否实现了 String() 方法，如果实现了，则直接打印输出 String() 方法的结果；

否则，会通过反射来遍历对象的成员进行打印。

### 函数调用

#### **C函数参数和返回值**

当我们在 x86_64 的机器上使用 C 语言中调用函数时，**参数都是通过寄存器和栈传递的**，其中：
六个以及六个以下的参数会按照顺序分别使用 edi、esi、edx、ecx、r8d 和 r9d 六个寄存器传递；
六个以上的参数会使用栈传递，函数的参数会以从右到左的顺序依次存入栈中；
**而函数的返回值是通过 eax 寄存器进行传递的，由于只使用一个寄存器存储返回值，所以 C 语言的函数不能同时返回多个值。**
优点: C语言的方式能够极大地减少函数调用的额外开销
缺点: 需要单独处理函数参数过多的情况

#### **Go函数参数和返回值**

我们发现 Go 语言**使用栈传递参数和接收返回值**，所以它只需要在栈上多分配一些内存就可以返回多个值。
缺点: CPU 访问栈的开销比访问寄存器高几十倍,函数入参和出参的内存空间需要在栈上进行分配,牺牲了函数调用的性能
优点: Go 语言的方式能够降低实现的复杂度并支持多返回值
优点: 不需要考虑超过寄存器数量的参数应该如何传递
优点: 不需要考虑不同架构上的寄存器差异

在调用函数之前会在栈上为返回值分配合适的内存空间，随后将入参从右到左按顺序压栈并拷贝参数，返回值会被存储到调用方预留好的栈空间上
因此,**Go 语言是值传递，无论是传递基本类型、结构体还是指针，都会对传递的参数进行拷贝**
所以将指针作为参数传入某一个函数时，在函数内部会对指针进行复制，也就是会同时出现两个指针指向原有的内存空间，所以 Go 语言中『传指针』也是传值。

在传递数组或者内存占用非常大的结构体时，我们在一些函数中应该尽量使用指针作为参数类型来避免发生大量数据的拷贝而影响性能

#### for和range

使用 for/range 的控制结构最终也会被 Go 语言编译器转换成经典三段循环

我们在遍历切片时给该切片追加的元素不会增加循环的执行次数，所以循环最终还是停了下来。

清空一个切片或者哈希表时,依次去遍历切片和哈希表,将元素置零
编译器优化会直接清空这片内存中的内容

使用 for range a {} 遍历数组和切片，不关心索引和数据的情况；
使用 for i := range a {} 遍历数组和切片，只关心索引的情况；
使用 for i, elem := range a {} 遍历数组和切片，关心索引和数据的情况；

**对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新的变量，在赋值的过程中就发生了拷贝，所以我们遍历的切片已经不是原始的切片变量了。**

而遇到这种同时遍历索引和元素的 range 循环时，**Go 语言会额外创建一个变量遍历存储切片中的元素，循环中使用的这个变量 v2 会在每一次迭代被重新赋值，在赋值时也发生了拷贝。**
**所以如果我们想要修改数组中元素，不应该直接获取 range 返回的变量，而应该使用a[index] 这种形式**

**使用 range 遍历哈希表时,每次遍历的顺序是随机的,**这是 Go 语言故意的设计，它在运行时为哈希表的遍历引入不确定性成一个随机数帮助我们随机选择一个桶开始遍历。Go 团队在设计哈希表的遍历时就不想让使用者依赖固定的遍历顺序，所以引入了随机数保证遍历的随机性。

遍历字符串时,只是在遍历时会获取字符串中索引对应的字节并将字节转换成 rune。我们在遍历字符串时拿到的值都是 rune 类型的变量

使用 range 遍历 Channel,该循环会使用 <-ch 从管道中取出等待处理的值，这个操作会阻塞当前的协程

```
for i := range ch {
}
```

#### make

make 的作用是**初始化内置的数据结构**，也就是我们在前面提到的**切片、哈希表和 Channel**；

```
slice := make([]int, 0, 100) //slice 是一个包含 data、cap 和 len 的私有结构体 internal/reflectlite.sliceHeader；
hash := make(map[int]bool, 10) //hash 是一个指向 runtime.hmap 结构体的指针；
ch := make(chan int, 5) //ch 是一个指向 runtime.hchan 结构体的指针；
```

相比与复杂的 make 关键字，new 的功能就很简单了，它只能接收一个类型作为参数然后返回一个指向该类型的指针：

new 的作用是根据传入的类型**在堆上分配一片内存空间并返回指向这片内存空间的指针**；

i := new(int) // 等价于var v int; i := &v
**使用 new 创建一个变量和先通过 var 初始化一个变量，然后对这个变量取地址没什么不同**，唯一的区别是，通过 new 函数不需要引入变量名称，所以使用上更加简洁、便利。

每次调用 new 函数都会返回唯一的地址变量,但是也会有例外，当定义一个空 struct 时，通过 new 创建一个变量时，返回的地址是相同的。



## 并发控制与同步

### context

#### 核心概念

`context` 几乎已经成为 Go 中**并发控制和超时控制**的标准方案。

- 用于取消、超时、截止时间控制
- 也可携带少量请求级元数据
- 并发安全

    ```
    type Context interface {
        Deadline() (deadline time.Time, ok bool) //该方法返回一个deadline和标识是否已设置deadline的bool值，如果没有设置												 				  deadline， 则ok == false，此时deadline为一个初始值的time.Time值
        Done() <-chan struct{} 				//该方法返回一个channel，需要在select-case语句中使用，如”case <-context.Done():”。
    							 				注意，这是一个只读的channel。 源码里没有地方会向这个 channel 里面塞入值,通过关闭这个通道来发							 					信号,因为读一个关闭的 channel 会读出相应类型的零值,因此在子协程里读这个 channel，除非被关								 				闭，否则读不出来任何东西。
        Err() error   						//该方法返回context关闭的原因。关闭原因由context实现控制，不需要用户设置。
        			  							比如Deadline context 关闭原因可能是因为deadline，也可能提前被主动关闭，那么关闭原因就会不同:
    				  							因deadline关闭：“context deadline exceeded”；
    				  							因主动关闭： “context canceled”。
    				  							当context关闭后，Err()返回context的关闭原因；当context还未关闭时，Err()返回nil；
    	Value(key interface{}) interface{} //用于在树状分布的goroutine间传递信息。
    									 		Value()方法根据key值查询map中的value。
    									 		设置值context.WithValue(ctx, Key, &v)
    									 		读取值v, ok := ctx.Value(Key).(*Values)
    }
    ```
    
    

#### 根上下文
空 context 用作根节点，本身不携带业务信息。

| 根 context | 用途 |
| --- | --- |
| `context.Background()` | 正式代码里常用的根 context |
| `context.TODO()` | 过渡阶段占位，表示“这里以后再补” |

context 包中常见实现有：

- `emptyCtx`
- `cancelCtx`
- `timerCtx`
- `valueCtx`

常见创建函数：

```
WithCancel()
WithDeadline()
WithTimeout()
WithValue()
```

#### cancelCtx


```
type cancelCtx struct {
    Context                        // 父上下文
    mu       sync.Mutex            // 互斥锁，并发安全
    done     chan struct{}         // 通道,一般经历如三个阶段：nil —> chan struct{} —> closed chan,c.done 是“懒汉式”创建，只有调用了 Done() 方法的时候才会被创建
    children map[canceler]struct{} // 以该上下文为父的的后代,其中key值即后代对象
    err      error                 // 默认是nil，在context被cancel时指定一个error变量： var Canceled = errors.New("context canceled")。
}
```

`cancelCtx` 的职责：

- 持有一个 `done` 通道
- 记录错误原因
- 维护子 context 集合

`cancel()` 调用后会：

- 关闭自己的 `done`通道
- 递归取消所有子节点
- 将自己从父节点的 children 中移除

`WithCancel()` 会返回：

- 一个新的 `cancelCtx`
- 一个 `cancel` 函数

timerCtx在cancelCtx基础上增加了deadline用于标示自动cancel的最终时间，和timer就是一个触发自动cancel的定时器。

#### timerCtx

```
type timerCtx struct {
    cancelCtx
    timer *time.Timer // 触发自动cancel的定时器
    deadline time.Time // 标示自动cancel的最终时间,WithDeadline()或WithTimeout()方法设置的
}
```

`timerCtx` 在 `cancelCtx` 基础上增加了：

- `deadline`
- `timer`

关闭原因：

- 主动取消：`context canceled`
- 超时/到期：`context deadline exceeded`

**创建流程**

- 判断父 context 的 deadline
- 创建 `timerCtx`
- 挂到父节点 children
- 启动定时器，到期自动取消

`WithTimeout()` 本质上就是 `WithDeadline()` 的语法封装。

#### valueCtx

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

`valueCtx` 只是在父 context 基础上挂了一组 `key-value`。

```
type valueCtx struct {
    Context // 父上下文
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

查找 value 时会沿着父链向上找。

- 当前节点找不到，就继续查父节点
- 一直找不到则返回 `nil`
- `valueCtx` 只负责传值，不负责取消

#### 使用建议
在真正使用 `Context.Value` 时要克制。

**适合放进 context 的内容**

- 认证令牌
- TraceID / RequestID
- 少量请求级元信息

**不适合放进 context 的内容**

- 普通函数参数
- 大对象
- 可选业务配置的大杂烩

**速记**

- context 主要做“控制”，传值只是辅助能力。

### 计时器

#### 底层结构

运行时会把定时器分桶管理：

- 一个桶对应一个或多个处理器 `P`
- 每个桶里维护定时器指针集合
- 多核机器上会尽量分散到不同桶中，减少竞争

```
type timer struct {
    tb *timersBucket //tb 就是用于存储当前定时器的桶
    i  int           //i 是当前定时器在堆中的索引,我们可以通过这两个变量找到当前定时器在堆中的位置
    when   int64     //当前定时器（Timer）被唤醒的时间
    period int64     //而 period 表示两次被唤醒的间隔,ticker计算器会使用该字段
    f      func(interface{}, uintptr) //每当定时器被唤醒时都会调用 f(args, now) 函数并传入 args 和当前时间作为参数
    arg    interface{}
}
```

对外暴露的定时器结构

```
type Timer struct {
    C <-chan Time   //当定时器到期时，就会被发送给当前定时器持有的 Channel C到期时间，订阅通道的 Goroutine 就会收到当前定时器到期的时间。
    r runtimeTimer  //就是上面的timer结构
}
```

Timer 定时器必须通过 NewTimer 或者 AfterFunc 函数进行创建

#### 创建方式
| API | 到期行为 |
| --- | --- |
| `time.NewTimer(d)` | 向 `Timer.C` 发送时间 |
| `time.AfterFunc(d, f)` | 到期后执行函数 `f` |

- 创建时会：
  - 选择一个合适的定时器桶
  - 把定时器插入四叉堆/最小堆结构
  - 必要时启动后台处理协程

创建时为当前定时器选择一个桶，我们会根据当前 Goroutine 所在处理器 P 的 id 选择一个合适的桶，随后将当前定时器加入桶中：
会先将最新加入的定时器加到队列的末尾,随后将当前定时器与四叉树（或者四叉堆）中的父节点进行比较和交换，**保证父节点的到期时间一定小于子节点：**
当当前定时器是第一个被加入四叉树的定时器，**我们还会通过 go timerproc(tb) 启动一个 Goroutine 用于监测处理当前树中的定时器：**

后台处理逻辑大致是：

- 没有定时器：休眠等待
- 最近的定时器未到期：继续等待
- 最近的定时器已到期：
  - 若是周期定时器：重设下一次触发时间
  - 若是一次性定时器：从堆中移除

#### 常见 API

| API | 作用 | 注意事项 |
| --- | --- | --- |
| `time.Sleep()` | 当前 Goroutine 休眠一段时间 | 底层也是定时器 |
| `time.NewTimer()` | 一次性定时器 | 可 `Stop` / `Reset` |
| `time.AfterFunc()` | 到期执行回调 | 适合延迟任务 |
| `time.NewTicker()` | 周期性触发 | 不用时一定 `Stop()` |
| `time.Tick()` | 返回只读 channel | 不能显式关闭，谨慎使用 |

- `Stop()` 会阻止后续触发。
- `Reset()` 一般用于已停止或已过期的定时器。

#### 性能与注意事项
性能

- 标准库定时器在高并发、短周期场景下误差会更明显。
- 特别是小于 `10ms` 的定时任务，触发时间通常只会晚不会早。
- 间隔越长，误差通常越小。

**速记**

- 高频短周期任务，不要过度迷信 `Timer/Ticker` 的精度。
- `NewTicker()` 用完记得 `Stop()`。

### sync包

#### 组件概览

`sync` 包中高频组件：

| 组件 | 用途 |
| --- | --- |
| `WaitGroup` | 等待一组 Goroutine 完成 |
| `Mutex` | 互斥锁 |
| `RWMutex` | 读写锁 |
| `Once` | 仅执行一次 |
| `Cond` | 条件变量 |
| `Map` | 并发安全 map |
| `Pool` | 对象复用池 |

#### WaitGroup
```
type WaitGroup struct {
    noCopy noCopy // noCopy 是 golang 源码中检测禁止拷贝的技术, 在编译期间会检查被拷贝的变量中是否包含 noCopy 或者 sync 关键字
	state1 uint64 // 高 32 位是计数器(代表目前尚未完成的个数。WaitGroup.Add(n) 将会导致 counter += n, 而 WaitGroup.Done() 将导致 counter--。)，
					 低 32 位是等待计数(代表目前已调用 WaitGroup.Wait 的 goroutine 的个数)
					 64位整数处理的原因是counter 和 waiter 在改变时需要保证并发安全,为了无锁优化
	state2 uint32 // 信号量,对应于 golang 中 runtime 内部的信号量的实现。WaitGroup 中会用到 sema 的两个相关函数，runtime_Semacquire 表示增加一个信号量，
	                 并挂起当前goroutine。 runtime_Semrelease 表示减少一个信号量，并唤醒 sema 上其中一个正在等待的 goroutine。
}
```

核心接口：

| 方法 | 作用 |
| --- | --- |
| `Add(delta)` | 增减计数器 |
| `Done()` | 等价于 `Add(-1)` |
| `Wait()` | 阻塞直到计数器归零 |

使用规则：

- `Add()` 要先于对应 Goroutine 启动或至少先于 `Wait()`。
- `Add()` 与 `Done()` 数量必须匹配。
- 计数器变成负数会 `panic`。
- `WaitGroup` 不应复制，函数传参通常传指针。

**速记**

- `wg.Add(1)` 写在 `go` 前面。
- `defer wg.Done()` 是最稳妥的写法。

#### Mutex
```
type Mutex struct {
    state int32 // 表示当前互斥锁的状态，包括锁定状态，饥饿模式标记，饥饿阈值1ms，互斥锁上等待的 Goroutine 个数
    sema  uint32 // 信号量，用来控制等待的goroutine 的阻塞，休眠，唤醒
}
```

Mutex 有两种主要模式：

| 模式 | 特点 |
| --- | --- |
| 正常模式 | 性能更好，存在竞争抢占 |
| 饥饿模式 | 更公平，优先把锁交给等待最久的 Goroutine |

- **等待时间较长时，可能切换到饥饿模式。**
- 正常模式下，锁竞争可能通过 CAS 和自旋优化。
- 自旋适合多核且短临界区场景。
- `Unlock()` 未配对会直接 `panic`。

**注意**

- Go 的 `Mutex` 不是可重入锁。
- 同一 Goroutine 重复加锁会死锁。

**速记**

- 临界区短：`Mutex` 常常足够。
- 避免“锁里再调会拿同一把锁”的代码结构。

#### RWMutex
RWMutex 适合**读多写少场景**。

| 操作组合 | 能否并发 |
| --- | --- |
| 读 + 读 | 可以 |
| 读 + 写 | 不可以 |
| 写 + 写 | 不可以 |

- `RLock/RUnlock` 用于读路径。
- `Lock/Unlock` 用于写路径。
- 写锁到来时，会阻止新的读锁继续进入。

**速记**

- 读多写少用 `RWMutex`。
- **写竞争重时，`RWMutex` 不一定比 `Mutex` 更快。**

#### sync.Once
- `Do(f)` 保证函数只执行一次。
- 即使 `f` 发生 `panic`，后续也不会再次执行。
- 多次传入不同函数，也只会执行第一次传入的那个。

**速记**

- 常用于单例初始化、延迟初始化、只关闭一次资源。

```
o := &sync.Once{}
for i := 0; i < 10; i++ {
    o.Do(func() {
        fmt.Println("only once")
    })
}

$ go run main.go

only once
```

#### sync.Cond
sync.Cond

- 用于“条件满足后再继续”的场景。
- 相比忙等 `for {}`，`Cond` 可以让 Goroutine 挂起等待，减少 CPU 空转。

| 方法 | 作用 |
| --- | --- |
| `Wait()` | 挂起等待，并在被唤醒后重新加锁 |
| `Signal()` | 唤醒一个等待者 |
| `Broadcast()` | 唤醒所有等待者 |

经典用法：

```go
c.L.Lock()
for !condition() {
    c.Wait()
}
// use condition
c.L.Unlock()
```

**速记**

- `Wait()` 前必须先加锁。
- 条件判断要放在 `for` 里，不要只用 `if`。

**FIFO 示例**


它提供了类似队列的 FIFO 的等待机制
先入先出队列(First Input First Output，FIFO)这是一种传统的按序执行方法，只能顺序写入数据，顺序的读出数据

```
l := sync.Mutex{}
fifo := &FIFO{
lock:  l,
cond:  sync.NewCond(&l),
queue: []int{},
}

func (f *FIFO) Offer(num int) error {
f.lock.Lock()
defer f.lock.Unlock()
f.queue = append(f.queue, num)
f.cond.Broadcast()
return nil
}

func (f *FIFO) Pop() int {
f.lock.Lock()
defer f.lock.Unlock()
for {
for len(f.queue) == 0 {
f.cond.Wait()
}
item := f.queue[0]
f.queue = f.queue[1:]
return item
}}
```

#### sync.Map

用一份适合并发无锁读取的只读快照 `read`，再配一份承接写入和更新的 `dirty`，让“读多写少”场景下的大部分读操作尽量不加锁

**适用场景**

- 原生 `map` 并发读写不安全。
- `sync.Map` **适合读多写少场景。**
- `sync.Map` 的零值可直接使用，但首次使用后不应复制。

**设计思路**

- **通过 `read` 和 `dirty` 两套 map 做读写分离。**
- 读路径尽量无锁，写路径按需加锁。
- 本质是**用空间换时间**。

**对比map + rwmutex**

每次读加 `RLock`，写加 `Lock`。

这没问题，但如果场景是：

- 读特别多
- 写很少
- key 一旦写入，后续多数只是读

那每次读都要碰锁，还是有成本。

所以 `sync.Map` 的优化目标是：

> **让绝大多数读请求走“无锁快路径”。**

**速记**

- 读多写少：`sync.Map` 可能合适。
- 写多：未必比 `map + RWMutex` 更好。

```
func main()  {
    var m sync.Map
    m.Store("qcrao", 18) // 1. 写入
    age, _ := m.Load("qcrao")     // 2. 读取
    // 3. 遍历
    m.Range(func(key, value interface{}) bool {
        name := key.(string)
        age := value.(int)
        fmt.Println(name, age)
        return true
    })
    m.Delete("qcrao")     // 4. 删除
}
```

 sync.Map 的数据结构

```
 type Map struct {
    mu Mutex        //dirty读写需要加锁, 将dirty更新到read，也需要加锁保护
    read atomic.Value // read 是 atomic.Value 类型，可以并发地读。只读区，原子可见，读时尽量不加锁
    dirty map[interface{}]*entry //dirty 是一个非线程安全的原始 map。脏数据区，写操作主要落这里，需要加锁
    misses int //每当从 read 中读取失败，都会将 misses 的计数值加 1，当加到一定阈值以后，需要将 dirty 提升为 read，以期减少 miss 的情形。
}
```

read 和 dirty 里存储的value是指针,read 和 dirty 各自维护一套 key，**key 指向的都是同一个 entry**

这样设计的好处是：

> **很多值更新，不需要改 map 结构本身(不会发生扩容)，只改 entry 内部状态即可。**

真正的 value 在 `entry` 里面**只要修改了这个 entry，对 read 和 dirty 都是可见的**



`read` 能无锁读，是因为它被设计成“几乎不被修改的只读快照”。



**核心操作**

- `Store`
  - 命中 `read`：直接更新
  - 未命中read：加锁后再看 `dirty`
  - 都没有：必要时创建 `dirty`
- `Load`
  - 先查 `read`
  - 再按需查 `dirty`
  - miss 太多会把 `dirty` 提升成 `read`
- `Delete`
  - 优先标记删除
  - 必要时从 `dirty` 真删
- `Range`
  - 遍历时可能先把 `dirty` 提升为 `read`

**store**

1. 如果在 read 里能够找到待存储的 key，并且对应的值没被标记删除时，直接更新对应的值即可。**因为read map是原子类型,因此并发安全**,read map和dirty map指向的是同一个值

2. 如果read 中没有这个 key,先加锁,再次向read查这个key(double check 可能上完锁就有了,确定上完锁后的状态),如果此刻read 中存在该 key,直接更新对应的 value。

   **为什么加锁后还要再查一次 read**

   因为在你准备加锁的这段时间里，可能别的 goroutine 已经把 dirty 提升成新的 read 了。

   所以要 double-check。

3. 如果 read 中还是没有此 key，那就查看 dirty 中是否有此 key，如果有，则直接更新对应的 value，这时 read 中还是没有此 key。更新 amended是否更新 字段，标识 dirty map 中存在而 read map 中没有

4. 如果 read 和 dirty 中都不存在该 key，则：
   a. 如果 dirty 为空，则需要创建 dirty，并从 read 中拷贝未被删除的元素；
   b. 更新 amended是否更新 字段，标识 dirty map 中存在而 read map 中没有；
   c. 将 k-v 写入 dirty map 中，read.m 不变

**Load:**
1.首先直接在 read 中找，如果找到了直接取出其中的值。
2.如果 read 中没有这个 key，
a.若 amended是否更新字段 为 fase，说明 dirty 没有更新的value，那直接返回 空和 false。
b.且 amended 为 true，说明 dirty 中可能存在我们要找的 key。上锁,再进行double check 的操作。若还是没有在 read 中找到，那么就从 dirty 中找。不管 dirty 中有没有找到，未击中计数器都将+1,表示一次未命中
如果 misses 未击中值达到阈值将 m.dirty 晋升为 read，并清空 dirty，清空 misses 计数值。这样，之前一段时间新加入的 key 都会进入到 read 中，从而能够提升 read 的命中率。

**Delete:**
先从 read 里查是否有这个 key，如果有则修改该值删除状态,并不是真正删除，这样 read 和 dirty 都能看到这个变化
如果没在 read 中找到这个 key，并且 dirty 不为空，那么就要操作 dirty 了，操作之前，还是要先上锁。然后进行 double check，如果仍然没有在 read 里找到此 key，则从 dirty 中删掉这个 key
原因在于，若两者都存在这个 key，仅做标记删除，可以在下次查找这个 key 时，命中 read，提升效率。若只有在 dirty 中存在时，read 起不到“缓存”的作用，直接删除。

**Range:**
Range 将遍历调用时刻 map 中的所有 k-v 对，将它们传给由使用者自己实现的f 函数，如果 f 返回 false，将停止遍历
遍历前判断dirty是否有新数据,有则将 dirty 提升为 read,之后遍历 read，取出中的值，调用 f(k, v)

**sync.map 没有 Len 方法,是因为计数会增加锁竞争**



#### sync.Pool

sync.Pool 是协程安全的。

**核心目标**

- 复用对象
- 降低频繁创建/销毁成本
- 减轻 GC 压力

使用前，需要初始化 Pool,设置好对象的 New 函数，用于在 Pool 里没有缓存的对象时，创建一个

```
type Person struct {
    Name string
}
pool = &sync.Pool {
        New: func() interface{} {
            fmt.Println("Creating a new Person")
            return new(Person)
        },
    }
```

```
//当调用 Get 方法时，如果池子里缓存了对象，就直接返回缓存的对象。如果没有存货，则调用 New 函数创建一个新的对象。
p1 := pool.Get().(*Person)
pool.Put(p1)
```

```
//Get 方法取出来的对象和上次 Put 进去的对象实际上是同一个，Pool 没有做任何“清空”的处理。但我们不应当对此有任何假设，因为在实际的并发使用场景中，无法保证这种顺序，并且GC可能会清除一些数据，最好的做法是在 Put 前，将对象清空。
如果Pool 里没有缓存的对象，会调用 New 创建一个
p2 := pool.Get().(*Person)
```

    // sync.Pool结构体
    type Pool struct {
        local     unsafe.Pointer // 指向GMP中的P的本地队列，实际类型为 [P]poolLocal切片
                                    访问时，P 的 id 对应 [P]poolLocal 下标索引，多个 goroutine 使用同一个 Pool 时，减少								了竞争，提升了性能。
        localSize uintptr         // [P]poolLocal的大小
    
    	New func() interface{} 	  // 自定义的对象创建回调函数，当 pool 中无可用对象时会调用此函数
    }



    type poolLocal struct {
        poolLocalInternal
    	// 用于补齐一个缓存行，让相关的字段能独立地加载到缓存行，cache line 是操作的最小单元，在 x86_64 体系下一般都是 64 字节
    	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
    }



    type poolLocalInternal struct {
        // P 的私有缓存区，使用时无需要加锁
        private interface{}
        // 公共缓存区,包含一个为单生产者、多消费者的无锁（atomic 实现）队列，可以动态增长。本地 P 可以 pushHead/popHead；其他 P 则只能 popTail
        shared  poolChain
    }



**Get / Put 流程**

- `Get()`
  - 先取当前 `P` 的私有缓存
  - 再取本地共享队列
  - 再尝试从其他 `P` 偷
  - 最后才调用 `New`
- `Put()`
  - 优先放回当前 `P` 的本地缓存
  - 放不下再放共享队列


**注意事项**

- `Pool` 不是无限缓存，GC 会清掉一部分对象。
- 取出的对象不保证就是上次放回去的那个。
- 放回前最好主动清空对象状态。
- `sync.Pool` 不适合做连接池，因为对象生命周期不受你完全控制。

**速记**

- `sync.Pool` 适合临时对象复用，不适合需要强生命周期管理的资源。

atomic.Value
`atomic.Value` 用来原子地存取任意类型的值。

| 方法 | 作用 |
| --- | --- |
| `Store(v)` | 原子写 |
| `Load()` | 原子读 |

**适合场景**

- 配置热更新
- 读多写少的共享快照
- 不想为简单读写引入锁竞争

**注意事项**

1. 不能存 `nil`
2. 同一个 `atomic.Value` 里不能混存不同动态类型
3. 存引用类型时要小心，外层原子不代表内部数据结构也并发安全

**速记**

- `atomic.Value` 适合“整块替换”，不适合复杂原地修改。

数据竞争和竞争条件
| 概念 | 含义 |
| --- | --- |
| 数据竞争 | 一个线程/协程读可变对象的同时，另一个在写 |
| 竞态条件 | 执行结果依赖不受控的时序 |

**避免数据竞争**

- `Mutex`
- `RWMutex`
- `sync/atomic`
- channel 通信

**降低竞态条件风险**

- 用 `WaitGroup` 控制完成顺序
- 用 `Context` 管理生命周期和超时
- 用更明确的同步协议代替“碰运气的执行顺序”



#### **乐观锁**

乐观锁是对于数据冲突保持一种乐观态度，操作数据时不会对操作的数据进行加锁（这使得多个任务可以并行的对数据进行操作），只有到数据提交的时候才通过一种机制来验证数据是否存在冲突
**(1)使用CAS理论来实现乐观锁**

CAS（Compare-And-Swap）

不加锁

修改前不阻塞

提交时用 CAS 检测冲突

实现原理就是：**更新之前先判断内存对象的值是否是获取的时候相同**，

- 如果相同，则认为没有更新过，更新，
- 如果不同，则认为更新了，重新获取数据并计算

Go `atomic` 包中的 CAS 是乐观锁的重要基础。

```
// CompareAndSwapUint32 executes the compare-and-swap operation for a uint32 value.
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
```


**CAS 缺陷**

- 高竞争下会频繁自旋，影响效率
- 无法天然解决 ABA 问题

**速记**

- `unsafe` 适合极少数性能敏感或底层场景。
- 绝大多数业务代码不该优先选 `unsafe`。

ABA问题是无锁结构实现中常见的一种问题，可基本表述为：
进程P1读取了一个数值A
P1被挂起(时间片耗尽、中断等)，进程P2开始执行
P2修改数值A为数值B，然后又修改回A

P1被唤醒，比较后发现数值A没有变化，程序继续执行。



**(2)version(也叫数据版本) 数据库中的乐观锁**
当线程A要更新数据值时，在读取数据的同时也会读取version值，在提交更新时，若刚才读取到的version值为当前数据库中的version值相等时才更新，否则重试更新操作，直到更新成功。



## 语言特性与工具

### defer

`defer` 会在当前函数返回前执行，常用于资源释放：

- 关闭文件
- 关闭连接
- 释放锁

**执行时机**

`return` 大致可理解为：

1. 给返回值赋值
2. 执行 defer
3. 真正返回

**执行顺序**

- 多个 `defer` 按**后进先出**执行。

**变量捕获**

| 方式 | 行为 |
| --- | --- |
| 作为参数传入 | defer 定义时就拷贝值 |
| 闭包引用外部变量 | defer 执行时读取当下值 |

**panic / recover**

- `panic` 会中断当前函数后续逻辑，**但仍会执行当前 Goroutine 上已注册的 defer。**
- `recover` 只能在 `defer` 中生效。

**速记**

- `defer` 常和资源释放、日志、恢复逻辑一起使用。
- 想拿“最终值”，通常要用闭包；想拿“当时值”，通常走参数。

```
defer func() {
            if err := recover(); err != nil {
                fmt.Println("recover success. err: ", err)
            }
        }()
```

这样就不会中止主程序了



### **闭包**

- 闭包可以**访问其外部函数中的变量**。

- 外部函数返回后，只要闭包还在引用这些变量，它们就可能继续存活。

- 好处：

  - 少用全局变量
  - 方便封装状态

- 风险：

  - 生命周期拉长
  - 不当使用可能增加内存占用

  

**类型转换速记**

| 类型          | 示例                      | 说明                 |
| ------------- | ------------------------- | -------------------- |
| 显式类型转换  | `uint(666)`               | 常规类型转换         |
| 类型断言      | `v, ok := x.(T)`          | 主要用于接口值       |
| `type switch` | `switch v := x.(type)`    | 判断接口动态类型     |
| `unsafe` 强转 | `(*T)(unsafe.Pointer(p))` | 底层指针操作，风险高 |

### 

### select

`select` 用于**同时等待多个 channel 的发送或接收**。

- 只有 channel 收发操作才能出现在 `case` 中。
- 如果没有可执行分支，`select` 会阻塞当前 Goroutine。

```
select {
case c <- x:
    x, y = y, x+y
case <-quit:
    fmt.Println("quit")
    return
}


for {
   select {
   }
}
```

**常见行为**

| 场景 | 行为 |
| --- | --- |
| 普通 `select` | 没有可执行分支时阻塞 |
| 带 `default` | 非阻塞 |
| `for + select` | 持续监听多个 channel |
| 没有 case | 永久阻塞 |
| case 中 channel 为 `nil` | 该 case 永远不会就绪 |

**调度特点**

- 多个 case 同时就绪时，运行时会**随机**选择一个，避免饥饿。
- 无可执行 case 时，**当前 Goroutine 会挂到相关 channel 的等待队列**。

**速记**

- 想持续监听：`for { select { ... } }`
- 想非阻塞尝试：加 `default`

### unsafe包

`unsafe` 允许绕过 Go 类型系统直接做底层内存操作。

- 任意类型指针都可以和 `unsafe.Pointer` 转换
- `unsafe.Pointer` 常与 `uintptr` 配合做地址运算

```
var i int8 = -1
var k uint8 = *(*uint8)(unsafe.Pointer(&i))

func bytes2string(b []byte) string{
    sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&b))

    sh := reflect.StringHeader{
        Data: sliceHeader.Data,
        Len:  sliceHeader.Len,
    }

    return *(*string)(unsafe.Pointer(&sh))
}
```

**核心概念**

| 类型 | 含义 |
| --- | --- |
| `unsafe.Pointer` | 通用指针 |
| `uintptr` | 仅保存地址数值，不具备指针语义 |

- `uintptr` 可以参与数学运算。
- `uintptr` 不是“活的指针”，GC 不会因为它而保活对象。
比如可以获取某个结构体中某个字段值
 s := make([]int, 9, 20)
 var Len = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(8)))
**风险**

- `uintptr` 不具备指针语义，相关对象可能被 GC 回收。
- `unsafe` 代码可读性差、可移植性差、容易引入未定义行为。

在map中定位key在哪个桶中,会用到指针运算
b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))

Offsetof方法可以获取成员偏移量
对于一个结构体，通过 offset 函数可以获取结构体成员的偏移量，通过指针运算,进而获取成员的地址，读写该地址的内存，就可以达到改变成员值的目的。

**通过 `unsafe` 相关函数，可以获取结构体私有成员地址。**



### 泛型

泛型让函数、类型、数据结构可以在保持类型安全的前提下处理多种类型。

**库代码、基础组件里更常用**：比如通用集合、缓存、队列、排序/比较工具、封装型数据结构，这类地方泛型收益很高。官方博客里也明确拿泛型树、带约束的接口来讲典型模式。

**业务代码里用得相对少一些**：尤其是 CRUD、服务编排、微服务接口层，很多团队还是更偏向直接写清楚具体类型，而不是为了“复用”硬抽一层泛型。这和 Go 一贯强调可读性、简单性很一致。

**核心概念**

| 名称 | 含义 |
| --- | --- |
| 类型形参 | 如 `T`、`K`、`V` |
| 类型实参 | 实际传入的类型，如 `int`、`string` |
| 类型约束 | 允许哪些类型参与实例化 |
| 泛型类型 | 带类型参数的类型定义 |
| 泛型函数 | 带类型参数的函数 |

**为什么用泛型**

- 复用逻辑而不丢失类型检查
- 避免为每种类型重复写一套代码
- 相比“接口 + 反射”更安全、性能更好

**和反射的对比**

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| 泛型 | 编译期检查、性能更稳、可读性更好 | 语法更复杂，需要设计约束 |
| 反射 | 动态能力强 | 可读性差、性能弱、类型错误更晚暴露 |

**基本语法**

```go
type Slice[T int | float32 | float64] []T
```

- `T` 是类型形参
- `int | float32 | float64` 是类型约束
- `Slice[T]` 是泛型类型

实例化示例：

```go
var a Slice[int] = []int{1, 2, 3}
```

类型参数也可以有多个：

```go
type MyMap[KEY int | string, VALUE float32 | float64] map[KEY]VALUE
```

举例:

```
type MyStruct[T int | string] struct {  
    Name string
    Data T
}

// 一个泛型接口(关于泛型接口在后半部分会详细讲解）
type IPrintData[T int | float32 | string] interface {
    Print(data T)
}

// 一个泛型通道，可用类型实参 int 或 string 实例化
type MyChan[T int | string] chan T

// 类型形参是可以互相套用的
type WowStruct[T int | float32, S []T] struct {
    Data     S
    MaxValue T
    MinValue T
}

// 定义泛型类型的时候，基础类型不能只有类型形参，如下：
// 错误，类型形参不能单独使用
type CommonType[T int|string|float32] T

// 当形参约束有*时会被编译器误认为表达式,因此是给类型约束包上 interface{} 或加上逗号消除歧义
type NewType2[T interface{*int|*float64}] []T

泛型类型可以互相嵌套
匿名结构体不支持泛型

/ 泛型接收器,使用泛型类型为接收者的方法
type Queue[T interface{}] struct {
    elements []T
}

// 将数据放入队列尾部
func (q *Queue[T]) Put(value T) {
    q.elements = append(q.elements, value)
}
使用时先实例化泛型类型,再调其方法

// 泛型函数,这种带类型形参的函数被称为泛型函数
func Add[T int | float32 | float64](a T, b T) T {
    return a + b
}
```

**常见能力**

| 能力 | 说明 |
| --- | --- |
| 泛型类型 | 可以定义带类型参数的结构体、切片、map、chan |
| 泛型函数 | 可以定义带类型参数的函数 |
| 泛型接收器 | 可以给泛型类型定义方法 |
| 自动推导类型实参 | 很多场景可省略显式类型参数 |

泛型函数调用示例：

```go
Add[float32](1.0, 2.0)
Add(1.0, 2.0)
```

**限制与注意**

- 匿名函数不支持泛型
- 方法本身不能单独声明类型参数，但可以依附于泛型接收器
- 匿名结构体不支持泛型
- 类型约束设计过宽或过长时，可读性会下降

**约束复用**

长类型约束可以提取成接口，便于维护：

```
type IntUintFloat interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | float32 | float64
}

type Slice[T IntUintFloat] []T
```

**`~` 的含义**

`~int` 表示：

- 不仅 `int` 可以用
- 底层类型是 `int` 的自定义类型也可以用

```
var s1 Slice[~int]
type MyInt int
var s2 Slice[MyInt]
```

go1.21 还为泛型 `slice` / `map` 场景补了不少常用工具能力。

**速记**

- 泛型优先解决“同一套逻辑适配多类型”的问题。
- 约束越精准，泛型越好维护。
- 能用泛型时，通常优先于反射。

### 反射

反射让程序在**运行时检查、修改、创建值和类型信息**。

**适用场景**

- 入参类型不固定
- 框架或通用库
- 标签解析、序列化、依赖注入等动态行为

**缺点**

- 可读性差
- 编译期类型检查弱化
- 性能明显低于普通代码

reflect 包里最核心的是：

| 类型/函数 | 作用 |
| --- | --- |
| `reflect.Type` | 描述类型信息 |
| `reflect.Value` | 描述值信息 |
| `reflect.TypeOf()` | 获取类型 |
| `reflect.ValueOf()` | 获取值 |

func TypeOf(i interface{}) Type  //返回一个接口，这个接口定义了一系列方法，利用这些方法可以获取关于类型的所有信息,调用此函数时，实参会先被转化为 interface{} 类型。这样，实参的类型信息、方法集、值信息都存储到 interface{} 变量里了。
使用反射来获取变量的类型： var t := reflect.Typeof (v) 该反射对象可以获取类型的方法,类型名称,子元素类型



    type Type interface {
        // 所有的类型都可以调用下面这些函数
    // 返回类型方法集里的第 `i` (传入的参数)个方法
    Method(int) Method
    
    // 通过名称获取方法
    MethodByName(string) (Method, bool)
    
    // 获取类型方法集里导出的方法个数
    NumMethod() int
    
    // 类型名称
    Name() string
    
    // 返回类型的字符串表示形式
    String() string
    
    // 返回类型的类型值
    Kind() Kind
    
    // 类型是否实现了接口 u
    Implements(u Type) bool
    
    // 是否可以赋值给 u
    AssignableTo(u Type) bool
    
    // 是否可以类型转换成 u
    ConvertibleTo(u Type) bool
    
    // 类型是否可以比较
    Comparable() bool
    
    // 下面这些函数只有特定类型可以调用
    // 如：Key, Elem 两个方法就只能是 Map 类型才能调用
    
    // 返回通道的方向，只能是 chan 类型调用
    ChanDir() ChanDir
    
    // 返回类型是否是可变参数，只能是 func 类型调用
    // 比如 t 是类型 func(x int, y ... float64)
    // 那么 t.IsVariadic() == true
    IsVariadic() bool
    
    // 返回内部子元素类型，只能由类型 Array, Chan, Map, Ptr, or Slice 调用
    Elem() Type
    
    // 返回结构体类型的第 i 个字段，只能是结构体类型调用
    // 如果 i 超过了总字段数，就会 panic
    Field(i int) StructField
    
    // 返回嵌套的结构体的字段
    FieldByIndex(index []int) StructField
    
    // 通过字段名称获取字段
    FieldByName(name string) (StructField, bool)
    
    // FieldByNameFunc returns the struct field with a name
    // 返回名称符合 func 函数的字段
    FieldByNameFunc(match func(string) bool) (StructField, bool)
    
    // 获取函数类型的第 i 个参数的类型
    In(i int) Type
    
    // 返回 map 的 key 类型，只能由类型 map 调用
    Key() Type
    
    // 返回 Array 的长度，只能由类型 Array 调用
    Len() int
    
    // 返回类型字段的数量，只能由类型 Struct 调用
    NumField() int
    
    // 返回函数类型的输入参数个数
    NumIn() int
    
    // 返回函数类型的返回值个数
    NumOut() int
    
    // 返回函数类型的第 i 个值的类型
    Out(i int) Type
    }


```
func ValueOf(i interface{}) Value //返回一个包含类型信息以及实际值的结构体变量，可以调用该反射对象的方法来获取和修改切片、map、结构体的字段值

Value 结构体定义了很多方法:

// 设置切片的 len 字段，如果类型不是切片，就会panic
 func (v Value) SetLen(n int)

 // 设置切片的 cap 字段
 func (v Value) SetCap(n int)

 // 设置字典的 kv
 func (v Value) SetMapIndex(key, val Value)

 // 返回切片、字符串、数组的索引 i 处的值
 func (v Value) Index(i int) Value

 // 根据名称获取结构体的内部字段值
 func (v Value) FieldByName(name string) Value
```


通过 `Type()`、`Interface()` 等方法，可以在 `interface`、`Type`、`Value` 之间转换。

**反射三大定理**

1. 从 `interface{}` 可以拿到 `Type` 和 `Value`
2. 从 `Value` 可以还原回 `interface{}`
3. 想修改原值，传入的值必须是可寻址、可设置的，通常需要指针

**速记**

- 能拿到值，不等于能修改值。
- 反射优先用于框架层，不优先用于普通业务逻辑。

## 运行时与内存

### 并发调度

**总览**

| 角色 | 含义 |
| --- | --- |
| `G` | Goroutine，待执行任务 |
| `M` | Machine，操作系统线程 |
| `P` | Processor，调度上下文和本地队列（调度能力（决定运行哪个 G）） |

**速记**

- M 必须绑定 P 才能执行 G。
- 本地队列、全局队列、工作窃取，是理解 GMP 的三个关键点。

**调度策略**

**调度策略简表**

- M优先从 `P` 的本地队列取任务
- 定期检查全局队列，避免任务饿死
- M绑定的P的本地没活时，从别的 `P` 偷任务

**系统调用期间的调度**

**分两种情况**

**情况1：普通 syscall（阻塞系统调用）**文件 IO、进程操作等子进程

- Goroutine 进入系统调用后，原线程 `M` 可能阻塞。
- 为了不让 `P` 空转，运行时会把 `P` 交给其他可用的 `M`。G 仍然绑定在 M 上不会被挂起
- P 找新的 M 继续执行
- 系统调用返回后：
  - 若有空闲 `P`，可继续执行G
  - 若没有，则将G回到全局队列等待调度

**情况2：网络 IO（netpoll / epoll）**，大部分 RPC / DB 驱动（如果是网络）

不进入真正阻塞 syscall,Go 会用：非阻塞 IO + epoll

如果没数据（EAGAIN）,G → 挂起（waiting）

M 继续执行,P 继续调度

**情况2： timer / channel / runtime 机制**

channel 阻塞

G 挂起，不阻塞 M

P 正常运行



**工作量窃取**

- 某个 `P` 空闲时，会尝试从其他 `P` 的本地队列窃取任务。
- 一般一次窃取一部分任务，而不是全部拿走。

**`GOMAXPROCS` 对性能的影响**

控制 P 的数量

| 场景 | 建议 |
| --- | --- |
| CPU 密集型 | 通常先设为 CPU 核数 |
| IO 密集型 | 可结合压测尝试更大值，**IO 密集型不是靠增加 P 提升性能** |

P 数量 = 并发执行能力
M 数量 ≥ P
G 数量 ≫ M

如果 P 太多，线程切换开销变大（反而更慢）

所以最佳：P = CPU 核数



**线程 vs Goroutine**

| 项 | 线程 | Goroutine |
| --- | --- | --- |
| 调度者 | 操作系统 | Go runtime |
| 初始栈 | 更大 | 更小，约 2KB 起步 |
| 切换成本 | 更高 | 更低 |
| 数量级 | 相对有限 | 可非常多 |

- 线程由操作系统调度，Goroutine 由 Go runtime 调度。
- Goroutine 栈更小、切换更轻，因此更适合高并发。

**G 状态速查**

| 状态 | 含义 |
| --- | --- |
| `_Gidle` | 刚分配未初始化 |
| `_Grunnable` | 可运行，等待调度 |
| `_Grunning` | 正在运行 |
| `_Gsyscall` | 正在系统调用 |
| `_Gwaiting` | 阻塞等待 |
| `_Gdead` | 已结束/可复用 |

**P 状态速查**

| 状态 | 含义 |
| --- | --- |
| `_Pidle` | 空闲 |
| `_Prunning` | 正在运行 |
| `_Psyscall` | 线程陷入系统调用 |
| `_Pgcstop` | 被 GC 停止 |
| `_Pdead` | 不再使用 |

**创建 Goroutine 时会做**

1. 获取或创建 `g`
2. 拷贝参数到新栈
3. 初始化上下文
4. 标记为 `_Grunnable`
5. 放入本地或全局队列

**常见触发调度的时机**

1. `go` 创建新 Goroutine
2. GC
3. 系统调用
4. 同步原语阻塞

**调度器找任务的顺序**

1. 本地运行队列
2. 全局运行队列
3. 从其他 `P` 窃取
4. 网络轮询器等其他来源

- Go 调度整体偏协作式，但 runtime 会用 `sysmon` 等机制做抢占辅助。
- 调度时会在不同来源中寻找可运行任务，保证公平性和吞吐。

**调度器初始化**

- 设置线程上限
- 创建并初始化 `P`
- 启动后台调度与 GC 相关任务

**g0**

- 每个线程都有自己的 `g0`
- 负责调度、栈增长、部分 runtime 管理工作
- 栈比普通 Goroutine 更大

**速记**

- 高并发的本质不是“没有线程”，而是“少量线程复用大量 Goroutine”。
- 系统调用、全局队列、工作窃取，是理解调度抖动的关键点。

### 内存管理

**总览**

 自动内存分配（allocator） + 垃圾回收（GC）

| 区域 | 特点 |
| --- | --- |
| 栈 | 编译器管理，生命周期通常跟函数调用一致，每个 goroutine 一个栈，初始：2KB，自动扩容 |
| 堆 | 运行时分配，GC 回收，大对象和逃逸对象（escape） |

| 类型     | 栈       | 堆     |
| -------- | -------- | ------ |
| 分配速度 | 快       | 慢     |
| 是否 GC  | ❌ 不需要 | ✅ 需要 |
| 生命周期 | 函数级   | 不确定 |

 **内存分配器（mheap / mspan / mcache）**

**Go 分配器核心思路**

- 按对象大小分类
- 使用多级缓存
- 优先本地分配，减少锁竞争

**分配器设计思路**

| 类型 | 优点 | 缺点 |
| --- | --- | --- |
| 线性分配器 | 快、实现简单 | 易碎片化 |
| 空闲链表分配器 | 可复用已回收块 | 遍历成本高 |

Go 的实现更接近“隔离适应”：

- 先按对象大小分类
- 不同大小走不同 `span class`
- 用多级缓存减少锁竞争

### 三层结构（重点）

```
mheap  （全局堆）
  ↓
mcentral（按 size class 管理）
  ↓
mcache（每个 P 私有缓存）
```

`mspan`  管理同一类对象的一组页

| 层级 | 作用 |
| --- | --- |
| `mcache` | 线程本地缓存，每个 P 一个 mcache，优先无锁分配 |
| `mcentral` | 多线程共享中心缓存 |
| `mheap` | 全局页堆，向操作系统申请内存 |

**缓存分层要点**

- `mcache` 绑定到执行上下文，尽量走本地缓存，减少锁竞争。
- `mcentral` 负责同类 `span` 的共享与回收，本质上是跨线程调配层。
- `mheap` 管全局页堆，空间不足时再向操作系统申请。

小对象分配路径：

- 先看 `mcache`
- 不够再找 `mcentral`
- 还不够再找 `mheap`
- 还不够向 OS 要

大对象（>32KB）

- 直接走 mheap

| 微对象 | `(0, 16B)` | tiny 分配器 |
| --- | --- | --- |
| 对象类型 | 大小范围 | 分配路径 |
| 小对象 | `[16B, 32KB]` | `mcache -> mcentral -> mheap` |
| 大对象 | `> 32KB` | 直接走 `mheap` |

**分配路径速记**

- 微对象：走 tiny 分配器
- 小对象：`mcache -> mcentral -> mheap`
- 大对象：直接走 `mheap`

**堆布局演进**

| 方案       | 特点                    | 问题 / 改进                             |
| ---------- | ----------------------- | --------------------------------------- |
| 早期连续堆 | 结构简单，地址映射直接  | 预留空间大，存在上限，和 C 混用容易冲突 |
| 稀疏堆     | 按 `heapArena` 分块管理 | 结构更复杂，但可扩展性更好              |

- Go 1.11 以后采用更偏稀疏的堆布局。
- 每个 `heapArena` 管理一块固定大小的地址空间，方便按需扩展。
- 稀疏布局减少了大块连续虚拟内存预留带来的限制。

**逃逸分析速查**

变量在栈上还是堆上

- 编译器会判断变量是否必须放到堆上。
- 典型原则：
  - 堆上的指针不能指向栈上已失效的对象
  - 返回局部变量地址、闭包持有变量等情况，常导致逃逸
- **逃逸到堆不代表错误，但会增加 GC 压力。**

**栈内存管理**

| 项       | 要点                             |
| -------- | -------------------------------- |
| 生命周期 | 通常跟随函数调用存在             |
| 分配方式 | 编译器和运行时配合管理           |
| 优势     | 分配释放快，额外元数据少         |
| 关键问题 | 栈空间不足时要扩容，过大时要缩容 |

**线程栈 vs Goroutine 栈**

| 项       | 线程栈       | Goroutine 栈       |
| -------- | ------------ | ------------------ |
| 初始大小 | 通常更大     | 初始更小，按需增长 |
| 扩容方式 | 多由系统控制 | runtime 动态扩缩容 |
| 成本     | 内存占用更高 | 更适合高并发       |

**分段栈 vs 连续栈**

| 方案 | 特点 | 评价 |
| --- | --- | --- |
| 分段栈 | 栈空间按段链接 | 容易出现热分裂问题 |
| 连续栈 | 扩容时复制到更大连续空间 | 现代 Go 使用，更稳定 |

- **连续栈扩容的关键，是把旧栈内容复制到新栈，并修正相关指针。**
- Go 现在采用连续栈，减少频繁扩缩容导致的抖动。

**栈扩缩容要点**

1. 函数调用前会检查剩余栈空间是否够用。
2. 不够时触发扩容，复制到更大的栈。
3. GC 期间若发现栈只用了较小比例，可能触发缩容。
4. 栈缩容不会无限减小，存在最小限制。

**性能观察**

- 栈内存通常不是主要瓶颈，**重点关注逃逸和堆分配**。
- 小对象频繁逃逸到堆，往往会比栈扩容更影响性能。

**内存分析与 pprof**

- `cpu`：CPU 使用热点
- `heap`：当前活跃对象
- `allocs`：累计分配历史
- `block`：阻塞等待
- `mutex`：锁竞争
- `goroutine`：协程栈与数量

**pprof 常见视角**

| 视角 | 含义 |
| --- | --- |
| `cpu` | CPU 消耗 |
| `heap` | 活跃对象内存 |
| `allocs` | 全部内存分配 |
| `block` | 阻塞分析 |
| `mutex` | 锁竞争分析 |
| `goroutine` | Goroutine 栈信息 |

```bash
go tool pprof --http=:8080 ~/Downloads/profile
```

**常见命令片段**

- `list <function regex>`：按函数名或正则查看文本明细
- `pprof.StartCPUProfile(...)` / `pprof.StopCPUProfile()`：采集 CPU profile
- `pprof.WriteHeapProfile(...)`：输出堆快照
- `profile` 和 `trace` 都需要采样一段时间再看结果

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

- `inuse_space`：看当前存活对象占用
- `alloc_objects`：看程序运行以来累计分配对象

```bash
go trace
```

**火焰图 / Web 查看**

```bash
pprof -http=:8080 cpu.prof
go tool pprof --http=:8080 ./pprof.mcu_main.samples.cpu.002.pb.gz
```

- `allocs` 看全部分配历史，`heap` 看当前活跃对象。
- Profiling 最好结合真实负载和压测一起看。

**速记**

- 小对象优先走本地缓存。
- 大对象直接走堆。
- 栈扩缩容通常不是首要瓶颈，逃逸和堆分配更值得先看。
- pprof 结果要在真实负载下看。

### 垃圾回收

**总览**

GC 负责回收堆上不再使用的对象。

| 策略 | 优点 | 缺点 | 代表 |
| --- | --- | --- | --- |
| 引用计数 | 回收及时 | 循环引用难处理、维护成本高 | Python / PHP / Swift |
| 标记-清除 | 能处理循环引用 | 需要标记和清扫阶段 | Go |
| 分代回收 | 吞吐通常更好 | 算法更复杂 | Java |

- Go 使用的是跟踪式 GC，核心是标记-清除及其并发变种。

**标记-清除流程**

1. 标记：从根对象出发，标记所有可达对象
2. 清除：回收未标记对象

- 标记阶段找存活对象，清理阶段回收垃圾对象。
- 传统标记-清除的问题是 STW 代价较大。

**三色抽象**

**可以“分阶段 + 并发”进行**
不需要一次性扫描所有对象

三色标记垃圾收集器的工作原理很简单，我们可以将其归纳成以下几个步骤：
1.从灰色对象的集合中选择一个灰色对象并将其标记成黑色；
2.将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收；
3.重复上述两个步骤直到对象图中不存在灰色对象；

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾

| 颜色 | 含义 |
| --- | --- |
| 白色 | 未访问（待回收） |
| 灰色 | 已发现但尚未扫描完成 |
| 黑色 | 已扫描并确认存活 |

**三色标记流程**

1. 从灰色集合取对象
2. 扫描它引用的对象，把后者染成灰色
3. 当前对象染成黑色
4. 重复直到没有灰色对象



Go 的 GC Roots 主要分为 4 大类：

- 每个 goroutine 的栈上的变量
- 全局变量（global variables）
- 寄存器中（还没写回栈）
- runtime 内部数据结构



#### 写屏障

因为用户程序可能在标记执行的过程中修改对象的指针，比如未被灰色引用的对象在后续被黑色对象应用，导致无法再被标记成灰色，而被清除，所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW

**为什么需要屏障**

- 并发标记期间，用户程序还在修改对象引用关系。
- 如果不加屏障，可能把本来还活着的对象误回收。

**三色不变性**

| 类型         | 含义                           |
| ------------ | ------------------------------ |
| 强三色不变性 | 黑色对象不能直接指向白色对象   |
| 弱三色不变性 | 黑到白仍需存在从灰色可达的路径 |

**常见写屏障**

| 屏障       | 核心思想                 | 特点             |
| ---------- | ------------------------ | ---------------- |
| 插入写屏障 | 新引用白对象时染灰       | 更保守           |
| 删除写屏障 | 删除引用时保护原对象链路 | 满足弱三色不变性 |
| 混合写屏障 | 同时结合插入和删除思路   | Go 方案          |

Go 语言中使用的两种写屏障技术，分别是 **Dijkstra 提出的插入写屏障**和 **Yuasa 提出的删除写屏障**

**插入写屏障**
在我们执行对象引用的时候，**如果被指针指向的被引用对象是白色的，那么插入写屏障函数会将该对象设置成灰色**

Dijkstra 的插入写屏障是一种相对保守的屏障技术，它会将有存活可能的对象都标记成灰色以满足强三色不变性
导致实际不再被引用的对象没有被回收，被错误标记的垃圾对象只有在下一个循环才会被回收

插入写屏障还有一个缺点是因为栈上的对象在垃圾收集中也会被认为是根对象，所以为了保证内存的安全，Dijkstra 必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象对象进行扫描，插入屏障是一个很耗费性能的行为，而栈需要更高的性能要求，因此，插入屏障技术只运用在堆内存空间里，不会运用到栈里。这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之前做出权衡。

**删除写屏障**
**当某个对象被删除引用时，如果自身为灰色或者白色，会被标记为灰色**，满足了弱三色不变式原则，**保护灰色对象到白色对象的可达路径不会断**。
使得后续如果有黑色对象要引用之前被删除引用的下游白色对象时，该白色对象因为是灰色的最后还是会被标记成黑色
删除写屏障也会导致实际不再被引用的对象没有被回收，被错误标记的垃圾对象只有在下一个循环才会被回收



**速记**

- Go GC 核心关键词：标记-清除、三色标记、写屏障、并发。
- 不该回收的对象被回收，会导致悬挂指针，是严重错误。

#### 1）并发 GC

👉 GC 和业务代码同时运行

#### 2）低 STW

👉 只在开始/结束短暂停顿

####  3）按比例触发

通过：

```
GOGC=100
```

👉 表示：

```
堆增长 100% → 触发 GC
```

**GC 触发与目标**

- `runtime.GC()` 可主动触发
- 运行时也会被动触发 GC
- `GOGC` 控制堆增长触发比例
- 并发 GC 还会用 pacing 算法控制触发时机

**GC 流程速查**

| 阶段 | 特点 |
| --- | --- |
| 清扫终止 | STW，准备并发标记 |
| 并发标记 | 与用户程序并发执行 |
| 标记终止 | STW，收尾标记 |
| 并发清理 | 回收空间、归还部分内存 |

**速记补充**

- 并发 GC 的核心难点不是“怎么扫”，而是“边扫边写时如何保持正确性”。
- 写屏障 + pacing + 并发标记，是现代 Go GC 的关键组合。

**主动 / 被动触发**

| 方式 | 说明 |
| --- | --- |
| 主动触发 | 调用 `runtime.GC()` |
| 被动触发 | 运行时按堆增长比例和系统监控触发 |

**分配过快时会怎样**

- 如果分配速度过快，GC 会让部分 Goroutine 承担 `Mark Assist`。
- 本质是“谁分配得快，谁帮忙标记”，用来控制堆继续失控增长。

**补充速记**

- `runtime.GC()` 是显式触发，不代表日常应该频繁手动调用。
- 调 GC 参数之前，先看对象分配模式和逃逸情况。

### 常用命令：`go build` 与 `go install`

go build：用于测试编译包，在项目目录下生成可执行文件（有main包）。

go install：主要用来生成库和工具。一是编译包文件（无main包），将编译后的包文件放到 pkg 目录下（$GOPATH/pkg）。二是编译生成可执行文件（有main包），将可执行文件放到 bin 目录（$GOPATH/bin）。

2. 相同点
都能生成可执行文件

3. 不同点
go build 不能生成包文件, go install 可以生成包文件
go build 生成可执行文件在当前目录下， go install 生成可执行文件在bin目录下（$GOPATH/bin）

make geth
会使用go run build/ci.go install ./cmd/geth

build/ci.go 会拼装go build参数为

/usr/local/go/bin/go build -ldflags "-X github.com/ethereum/go-ethereum/internal/version.gitCommit=b818e73ef39e376bd5c6d9074c0a432301042e3b -X github.com/ethereum/go-ethereum/internal/version.gitDate=20221220 -extldflags '-Wl,-z,stack-size=0x800000'" -tags urfave_cli_no_docs -trimpath -v -o /home/go-ethereum/build/bin/geth ./cmd/geth

## 网络编程

### Go网络编程

`net` 包是 Go 网络编程的基础入口。

**核心接口**

| 接口 | 作用 | 常见协议 |
| --- | --- | --- |
| `Conn` | 面向连接的数据收发接口 | TCP、Unix、部分 IP/UDP 场景 |
| `Listener` | 监听流式连接 | TCP、Unix |
| `PacketConn` | 面向数据报的收发接口 | UDP、IP |

**快速理解**

- `Conn` 更像“已经建立好的连接”
- `Listener` 更像“服务端监听入口”
- `PacketConn` 更像“按数据包收发”

**Dial / Listen 速查**

| API | 用途 |
| --- | --- |
| `net.Dial()` | 客户端发起连接 |
| `net.Listen()` | 服务端监听流式协议 |
| `net.ListenUDP()` | 服务端监听 UDP |

**协议差异**

| 协议 | 是否面向连接 | 是否有 `Accept()` |
| --- | --- | --- |
| TCP | 是 | 有 |
| UDP | 否 | 没有 |

通常业务中直接用 TCP / UDP 即可；直接操作 IP 层时，要自己处理更底层的数据包细节。

TCP服务端程序的处理流程：
1. 监听端口：`net.Listen("tcp", addr)`
2. 接收连接：`listener.Accept()`
3. 启动 Goroutine 处理连接

TCP客户端发送流程
**TCP 客户端流程**

```
// 建立与服务端的链接 
conn, err := net.Dial("tcp", "127.0.0.1:20000")
// 进行数据收发 	
_, err = conn.Write([]byte(inputInfo))  n, err := conn.Read(buf[:])
// 关闭链接 
defer conn.Close()
```

TCP粘包
因为 TCP 是字节流协议，长连接下可能出现“粘包 / 拆包”问题。

**常见原因**

- 发送端：Nagle 算法合并发送
- 接收端：应用层读取不及时，缓冲区积压多段数据

**解决思路**

- 关键是让接收方知道“一个消息边界在哪里”
- 常见做法：
  - 固定长度协议
  - 长度字段 + 包体
  - 特殊分隔符

**速记**

- TCP 没有天然消息边界，应用层必须自己定义协议。

UDP服务端流程
**UDP 服务端流程**

```
// 监听端口
listen, err := net.ListenUDP("udp", &net.UDPAddr{
		    IP:   net.IPv4(0, 0, 0, 0),
		    Port: 30000,
 })
// 接受数据 
n, addr, err := listen.ReadFromUDP(data[:]) // 接收数据
// 发送数据 
_, err = listen.WriteToUDP(data[:n], addr) // 发送数据
```

**TCP vs UDP 速查**

| 项 | TCP | UDP |
| --- | --- | --- |
| 连接方式 | 面向连接 | 无连接 |
| 数据形式 | 字节流 | 数据报 |
| 是否可靠 | 可靠 | 不保证可靠 |
| 是否有粘包问题 | 有 | 通常没有 |
| 常见接口 | `Dial` / `Listen` / `Accept` | `DialUDP` / `ListenUDP` / `ReadFromUDP` |

**速记**

- TCP 适合可靠传输。
- UDP 适合轻量、低时延、按包处理的场景。
- 做 TCP 协议时，一定先想清楚消息边界。

