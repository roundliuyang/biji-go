# Go并发编程



## 同步原语的适用场景

- 共享资源。并发地读写共享资源，会出现数据竞争（data race）的问题，所以需要  Mutex、RWMutex 这样的并发原语来保护。
- 任务编排。需要 goroutine 按照一定的规律执行，而 goroutine 之间有相互等待或者依 赖的顺序关系，我们常常使用 WaitGroup 或者 Channel 来实现。
- 消息传递。信息交流以及不同的 goroutine 之间的线程安全的数据交流，常常使用  Channel 来实现。



## Mutex: 4种易错场景大盘点



### Lock/Unlock 不是成对出现

Lock/Unlock 没有成对出现，就意味着会出现死锁的情况，或者是因为 Unlock 一个未加 锁的 Mutex 而导致 panic。



### Copy 已使用的 Mutex

Package sync 的同步原语在使用后是不能复制的。我们知道 Mutex 是最常用 的一个同步原语，那它也是不能复制的。

原因在于，Mutex 是一个有状态的对象，它的 state 字段记录这个锁的状态。如果你要复 制一个已经加锁的 Mutex 给一个新的变量，那么新的刚初始化的变量居然被加锁了，这显 然不符合你的期望，因为你期望的是一个零值的 Mutex。关键是在并发环境下，你根本不 知道要复制的 Mutex 状态是什么，因为要复制的 Mutex 是由其它 goroutine 并发访问 的，状态可能总是在变化。 

当然，你可能说，你说的我都懂，你的警告我都记下了，但是实际在使用的时候，一不小 心就踩了这个坑，我们来看一个例子。

```go
type Counter struct {
	sync.Mutex
	Count int
}

func main() {
	var c Counter
	c.Lock()
	defer c.Unlock()
	c.Count++
	foo(c) // 复制锁
}

// 这里Counter的参数是通过复制的方式传入的

func foo(c Counter) {
	c.Lock()
	defer c.Unlock()
	fmt.Println("in foo")
}
```

在调用 foo 函数的时候，调用者会复制 Mutex 变量 c 作为 foo 函数的参数，不 幸的是，复制之前已经使用了这个锁，这就导致，复制的 Counter 是一个带状态  Counter。



### 重入

Mutex 不是可 重入的锁。

想想也不奇怪，因为 Mutex 的实现中没有记录哪个 goroutine 拥有这把锁。理论上，任 何 goroutine 都可以随意地 Unlock 这把锁，所以没办法计算重入条件，毕竟，“臣妾做 不到啊”！

所以，一旦误用 Mutex 的重入，就会导致报错。下面是一个误用 Mutex 的重入例子：

```java
func foo(l sync.Locker) {
	fmt.Println("in foo")
	l.Lock()
	bar(l)
	l.Unlock()
}

func bar(l sync.Locker) {
	l.Lock()
	fmt.Println("in bar")
	l.Unlock()
}

func main() {
	l := &sync.Mutex{}
	foo(l)
}
```

写完这个 Mutex 重入的例子后，运行一下，你会发现类似下面的错误。程序一直在请求 锁，但是一直没有办法获取到锁，结果就是 Go 运行时发现死锁了，没有其它地方能够释 放锁让程序运行下去，你通过下面的错误堆栈信息就能定位到哪一行阻塞请求锁。



学到这里，你可能要问了，虽然标准库 Mutex 不是可重入锁，但是如果我就是想要实现一 个可重入锁，可以吗？ 可以，那我们就自己实现一个。这里的关键就是，实现的锁要能记住当前是哪个  goroutine 持有这个锁。我来提供两个方案。

- 方案一：通过 hacker 的方式获取到 goroutine id，记录下获取锁的 goroutine id，它 可以实现 Locker 接口。
- 方案二：调用 Lock/Unlock 方法时，由 goroutine 提供一个 token，用来标识它自 己，而不是我们通过 hacker 的方式获取到 goroutine id，但是，这样一来，就不满足  Locker 接口了。

可重入锁（递归锁）解决了代码重入或者递归调用带来的死锁问题，同时它也带来了另一 个好处，就是我们可以要求，只有持有锁的 goroutine 才能 unlock 这个锁。这也很容易 实现，因为在上面这两个方案中，都已经记录了是哪一个 goroutine 持有这个锁。

下面我们具体来看这两个方案怎么实现。



#### 方案一：goroutine id

这个方案的关键第一步是获取 goroutine id，方式有两种，分别是简单方式和 hacker 方 式。 简单方式，就是通过 runtime.Stack 方法获取栈帧信息，栈帧信息里包含 goroutine id。 你可以看看上面 panic 时候的贴图，goroutine id 明明白白地显示在那里。



了解了简单方式，接下来我们来看 hacker 的方式，这也是我们方案一采取的方式。 

首先，我们获取运行时的 g 指针，反解出对应的 g 的结构。每个运行的 goroutine 结构的  g 指针保存在当前 goroutine 的一个叫做 TLS 对象中。 

第一步：我们先获取到 TLS 对象； 

第二步：再从 TLS 中获取 goroutine 结构的 g 指针； 

第三步：再从 g 指针中取出 goroutine id。



#### 方案二：token

方案一是用 goroutine id 做 goroutine 的标识，我们也可以让 goroutine 自己来提供标 识。不管怎么说，Go 开发者不期望你利用 goroutine id 做一些不确定的东西，所以，他 们没有暴露获取 goroutine id 的方法。

下面的代码是第二种方案。调用者自己提供一个 token，获取锁的时候把这个 token 传 入，释放锁的时候也需要把这个 token 传入。通过用户传入的 token 替换方案一中  goroutine id，其它逻辑和方案一一致。



### 死锁

两个或两个以上的进程（或线程，goroutine）在执行过程中，因 争夺共享资源而处于一种互相等待的状态，如果没有外部干涉，它们都将无法推进下去， 此时，我们称系统处于死锁状态或系统产生了死锁。 我们来分析一下死锁产生的必要条件。如果你想避免死锁，只要破坏这四个条件中的一个 或者几个，就可以了。

1. 互斥： 至少一个资源是被排他性独享的，其他线程必须处于等待状态，直到资源被释 放。 
2. 持有和等待：goroutine 持有一个资源，并且还在请求其它 goroutine 持有的资源，也 就是咱们常说的“吃着碗里，看着锅里”的意思。 
3. 不可剥夺：资源只能由持有它的 goroutine 来释放。
4. 环路等待：一般来说，存在一组等待进程，P={P1，P2，…，PN}，P1 等待 P2 持有的 资源，P2 等待 P3 持有的资源，依此类推，最后是 PN 等待 P1 持有的资源，这就形成 了一个环路等待的死结。



