---
title: "Rust：异步编程 + Tokio"
date: 2026-04-12T12:57:10+08:00
draft: false
tags: ["Rust"]
categories: ["Rust"]
summary: "本文系统讲解Rust异步编程原理与Tokio运行时，涵盖async/await机制、Future状态机实现、Waker唤醒模型、执行器从忙轮询到按需唤醒的演进，以及Tokio适用场景与最佳实践。"
ShowToc: true
TocOpen: false
---

> 很久没更新 Rust 相关的文章了，上次更新是 2025-09-27。随着 AI Vide Coding 的爆发，真没啥动力继续更新 Rust 相关的文章了。

> 不过最近，根据周围人的亲身 Coding 经历，我逐渐意识到可能 Rust 是 Vide Coding 时代最有性价比的语言了。Rust 极陡峭的学习曲线对 AI 来说不值一提，AI 非常擅长处理那些繁琐的生命周期标注、所有权规则和类型体操。Rust的严格编译器对 AI 来说是巨大的优势——编译器就是最好的 AI 校验器。并且最重要的是，Rust 的性能是免费的。AI 写 Python 和写 Rust 工作量几乎没差别，但产出性能可能差一个数量级。

> 所以朝花夕拾，有始有终，硬着头皮继续更新相关文章吧。

## 异步（Async）编程介绍

异步编程允许我们同时并发运行大量的任务，却仅仅需要几个甚至一个 OS 线程或 CPU 核心。

`async` 和多线程都可以实现并发编程，后者甚至还能通过线程池来增强并发能力，但是这两个方式并不互通，从一个方式切换成另一个需要大量的代码重构工作，因此提前为自己的项目选择适合的并发模型就变得至关重要。

- OS 线程非常适合少量任务并发，因为线程的创建和上下文切换是非常昂贵的，甚至于空闲的线程都会消耗系统资源。

- 对于长时间运行的 CPU 密集型任务，例如并行计算，使用线程将更有优势。这种密集任务往往会让所在的线程持续运行，任何不必要的线程切换都会带来性能损耗，因此高并发反而在此时成为了一种多余。

- 而高并发更适合 IO 密集型任务，例如 web 服务器、数据库连接等网络服务，因为这些任务绝大部分时间都处于等待状态，如果使用多线程，那线程大量时间会处于无所事事的状态，再加上线程上下文切换的高昂代价，让多线程做 IO 密集任务变成了一件非常奢侈的事。

而使用 `async`，既可以有效的降低 CPU 和内存的负担，又可以让大量的任务并发的运行，一个任务一旦处于 IO 或者其他等待（阻塞）状态，就会被立刻切走并执行另一个任务，而这里的任务切换的性能开销要远远低于使用多线程时的线程上下文切换。

事实上，`async` 底层也是基于线程实现，但是它基于线程封装了一个运行时，可以将多个任务映射到少量线程上，然后将线程切换变成了任务切换，后者仅仅是内存中的访问，因此要高效的多。

总之，async 编程并没有比多线程更好，最终还是根据你的使用场景作出合适的选择，如果无需高并发，或者也不在意线程切换带来的性能损耗，那么多线程使用起来会简单、方便的多！最后再简单总结下：

- 有大量 IO 任务需要并发运行时，选 `async` 模型

- 有部分 IO 任务需要并发运行时，选多线程，如果想要降低线程创建和销毁的开销，可以使用线程池

- 有大量 CPU 密集任务需要并行运行时，例如并行计算，选多线程模型，且让线程数等于或者稍大于 CPU 核心数

- 无所谓时，统一选多线程

### async 和多线程的性能对比

| 操作 | async | 线程 |
| --- | --- | --- |
| 创建 | 0.3 微秒 | 17 微秒 |
| 线程切换 | 0.2 微秒 | 1.7 微秒 |

## 异步（Async）编程基础

```rust
// `block_on`会阻塞当前线程直到指定的`Future`执行完成，这种阻塞当前线程以等待任务完成的方式较为简单、粗暴，
// 好在其它运行时的执行器(executor)会提供更加复杂的行为，例如将多个`future`调度到同一个线程上执行。
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // 返回一个Future, 因此不会打印任何输出
    block_on(future); // 执行`Future`并等待其运行完成，此时"hello, world!"会被打印输出
}
```

这段代码展示了异步编程最基本的模式：`async fn` 调用后并不会立即执行，而是返回一个 `Future`。Future 本身只是一个"待执行的计划"，必须交给执行器才能真正运行。这里的 `block_on` 就是最简单的执行器——它会阻塞当前线程，不断驱动 Future 直到完成

### 在 async fn 中调用另一个 async fn

如果你要在一个 `async fn` 函数中去调用另一个 `async fn` 并等待其完成后再执行后续的代码，该如何做？例如：

```rust
use futures::executor::block_on;

async fn hello_world() {
    hello_cat(); // 将不会执行
    println!("hello, world!");
}

async fn hello_cat() {
    println!("hello, kitty!");
}

fn main() {
    let future = hello_world();
    block_on(future);
}
```

输出：

```
warning: unused implementer of `futures::Future` that must be used
 --> src/main.rs:6:5
  |
6 |     hello_cat();
  |     ^^^^^^^^^^^^
= note: futures do nothing unless you `.await` or poll them
...
hello, world!
```

编译器的警告信息说得很清楚：`futures do nothing unless you .await or poll them`。直接调用 `hello_cat()` 只是创建了一个 Future 并立刻丢弃，函数体根本没有执行。要让它真正运行，需要使用 `.await` 来驱动这个 Future 并等待其完成。

```rust
use futures::executor::block_on;

async fn hello_world() {
    hello_cat().await;
    println!("hello, world!");
}

async fn hello_cat() {
    println!("hello, kitty!");
}

fn main() {
    let future = hello_world();
    block_on(future);
}
```

### 并发执行两个 Future

前面的 `.await` 是串行的——必须等上一个 Future 完成才能执行下一个。但很多时候我们希望多个任务同时推进，例如一边下载数据一边渲染 UI。`futures::join!` 宏可以做到这一点：它接收多个 Future，在同一个线程上交替 poll 它们，哪个能推进就推进哪个，从而实现并发。

```rust
async fn async_main() {
    let f1 = learn_and_sing();
    let f2 = dance();

    // `join!`可以并发的处理和等待多个`Future`，若`learn_and_sing Future`被阻塞，
    // 那`dance Future`可以拿过线程的所有权继续执行。若`dance`也变成阻塞状态，
    // 那`learn_and_sing`又可以再次拿回线程所有权，继续执行。
    // 若两个都被阻塞，那么`async main`会变成阻塞状态，然后让出线程所有权，
    // 并将其交给`main`函数中的`block_on`执行器
    futures::join!(f1, f2);
}

fn main() {
    block_on(async_main());
}
```

## 异步原理：从零构建运行时

前面我们学会了 `async`/`await` 的基本用法，接下来深入底层，理解 Rust 异步运行时的核心机制。本章的脉络如下：

1. 先用一个简化版 Future 建立直觉（`SimpleFuture`）

2. 过渡到 Rust 标准库中真实的 Future（`Pin`、`Context`）

3. 亲手实现一个自定义 Future，并看看编译器如何将 async fn 变成状态机

4. 从零构建执行器，经历V1 忙轮询 → V2 按需唤醒的演进，理解 `Waker` 的意义

5. 最后把所有组件串起来，看完整的运行流程

### Future 特征（简化版）

Future 的定义：它是一个能产出值的异步计算。我们先用一个简化版的 trait 来理解核心思想：

```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

Future 需要被执行器 `poll`（轮询）后才能运行，通过调用该方法，可以推进 Future 的进一步执行，直到被切走为止。

在当前 `poll` 中，Future 可以被完成，则会返回 `Poll::Ready(result)`，反之则返回 `Poll::Pending`，并且安排一个 `wake` 函数：当未来 Future 准备好进一步执行时，该函数会被调用，然后管理该 Future 的执行器会再次调用 `poll` 方法，此时 Future 就可以继续执行了。

考虑一个需要从 socket 读取数据的场景：如果有数据，可以直接读取数据并返回 `Poll::Ready(data)`，但如果没有数据，Future 会被阻塞且不会再继续执行，此时它会注册一个 `wake` 函数，当 socket 数据准备好时，该函数将被调用以通知执行器：我们的 Future 已经准备好了，可以继续执行。

下面的 `SocketRead` 结构体就是一个 Future：

```rust
    socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // socket有数据，写入buffer中并返回
            Poll::Ready(self.socket.read_buf())
        } else {
            // socket中还没数据
            //
            // 注册一个`wake`函数，当数据可用时，该函数会被调用，
            // 然后当前Future的执行器会再次调用`poll`方法，此时就可以读取到数据
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```

### 通过状态机实现并发 Future

如果需要同时运行多个 Future 或链式调用多个 Future，也可以通过无内存分配的状态机实现：

```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}

/// 一个SimpleFuture，它会并发地运行两个Future直到它们完成
///
/// 之所以可以并发，是因为两个Future的轮询可以交替进行，一个阻塞，另一个就可以立刻执行，反之亦然
pub struct Join<FutureA, FutureB> {
    // 结构体的每个字段都包含一个Future，可以运行直到完成.
    // 等到Future完成后，字段会被设置为 `None`. 这样Future完成后，就不会再被轮询
    a: Option<FutureA>,
    b: Option<FutureB>,
}

impl<FutureA, FutureB> SimpleFuture for Join<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        // 尝试去完成一个 Future `a`
        if let Some(a) = &mut self.a {
            if let Poll::Ready(()) = a.poll(wake) {
                self.a.take();
            }
        }

        // 尝试去完成一个 Future `b`
        if let Some(b) = &mut self.b {
```

### 从 SimpleFuture 到 Rust 真实的 Future

前面的 `SimpleFuture` 帮助我们理解了核心思想，但 Rust 标准库中真实的 `Future` trait 有两个关键区别：

```rust
trait Future {
    type Output;
    fn poll(
        // 1. `self`的类型从`&mut self`变成了`Pin<&mut Self>`:
        //    Pin 保证 Future 在内存中不会被移动，这对包含自引用的异步状态机至关重要
        self: Pin<&mut Self>,
        // 2. `wake: fn()` 变成了 `cx: &mut Context<'_>`:
        //    Context 内部包含一个 Waker，不仅能唤醒任务，还能携带额外的调度信息
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

从这里开始，后续所有代码都将使用这个真实的 `Future` trait。

### 实现自定义 Future：Delay

下面来实现一个具体的 Future，它将：1. 等待某个特定时间点的到来 2. 在标准输出打印文本 3. 生成一个字符串

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

// 为我们的 Delay 类型实现 Future 特征
impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            // 时间到了，Future 可以结束
            println!("Hello world");
            // Future 执行结束并返回 "done" 字符串
            Poll::Ready("done")
        } else {
            // 目前先忽略下面这行代码
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    // 运行并等待 Future 的完成
    let out = future.await;
```

注意这里的 `cx.waker().wake_by_ref()` —— 它在返回 `Pending` 的同时立刻通知执行器"我还没好，但你可以马上再来问我"。这本质上是一种忙轮询，后面我们会看到它的问题以及如何改进。

### 编译器如何处理 async fn

我们已经知道 `async fn` 不会立即执行，它返回一个 Future：

```rust
use tokio::net::TcpStream;

async fn my_async_fn() {
    println!("hello from async");
    let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
    println!("async TCP operation complete");
}

#[tokio::main]
async fn main() {
    let what_is_this = my_async_fn();
    // 上面的调用不会产生任何效果

    // ... 执行一些其它代码

    what_is_this.await;
    // 直到 .await 后，文本才被打印，socket 连接也被创建和关闭
}
```

那编译器是怎么做到的？秘密在于：编译器会将 `async fn` 的函数体编译成一个枚举状态机。每个 `.await` 点就是一个状态分界。

以前面使用 `Delay` 的 `main` 函数为例，编译器生成的代码类似：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    // 初始化，但永远不会被 poll
    State0,
    // 等待 `Delay` 运行，例如 `future.await` 代码行
    State1(Delay),
    // Future 执行完成
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    let when = Instant::now() +
                        Duration::from_millis(10);
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
```

编译器会将 Future 变成状态机，其中 `MainFuture` 包含了 Future 可能处于的状态：从 `State0` 状态开始，当 `poll` 被调用时，Future 会尝试去尽可能的推进内部的状态，若它可以被完成时，就会返回 `Poll::Ready`，其中还会包含最终的输出结果

这就是 async/await 的零成本抽象——没有堆分配，没有动态调度，只是一个普通的 `enum` + `loop` + `match`。

![](images/img_01.png)

### 构建执行器

`async fn` 返回 Future，而 Future 是惰性的，需要一个执行器（Executor） 来不停地 `poll` 推动它们直到完成。接下来我们从零构建一个执行器，经历两个版本的演进。

#### V1：忙轮询

最简单的执行器实现：用一个 `VecDeque` 存放所有任务，不断取出来 `poll`，没完成就塞回去：

```rust
fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }

    /// 生成一个 Future并放入 mini-tokio 实例的任务队列中
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }

    fn run(&mut self) {
```

这个执行器能跑，但有个严重问题：它不停地 poll 所有任务，即使绝大部分 Future 并没有准备好。这就像老板每 5 秒钟就问你"做完了吗？"——巨大的 CPU 浪费。

我们需要的是一种**"通知 → 运行"**机制：Future 在无法继续时安静等待，一旦就绪主动通知执行器，执行器再去 poll。这就是 `Waker` 存在的意义。

#### 引入 Waker：从忙轮询到按需唤醒

回顾 `Future::poll` 的签名：

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context)
    -> Poll<Self::Output>;
```

`Context` 参数中包含 `waker()` 方法，返回一个绑定到当前任务上的 `Waker`。`Waker` 上定义了 `wake()` 方法，用于通知执行器：我准备好了，可以再来 poll 我了。

有了 `Waker`，Future 就不需要忙轮询了。以定时器为例，我们可以实现一个 `TimerFuture`：在 `poll` 返回 `Pending` 时将 `Waker` 存下来，然后启动一个计时线程，等时间到了由计时线程调用 `waker.wake()` 来通知执行器。

首先定义共享状态和 Future 结构体：

```rust
pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

/// 在Future和等待的线程间共享状态
struct SharedState {
    /// 定时(睡眠)是否结束
    completed: bool,

    /// 当睡眠结束后，线程可以用`waker`通知`TimerFuture`来唤醒任务
    waker: Option<Waker>,
}
```

Future 的具体实现：

```rust
impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 通过检查共享状态，来确定定时器是否已经完成
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            // 设置`waker`，这样新线程在睡眠(计时)结束后可以唤醒当前的任务，接着再次对`Future`进行`poll`操作,
            //
            // 下面的`clone`每次被`poll`时都会发生一次，实际上，应该是只`clone`一次更加合理。
            // 选择每次都`clone`的原因是： `TimerFuture`可以在执行器的不同任务间移动，如果只克隆一次，
            // 那么获取到的`waker`可能已经被篡改并指向了其它任务，最终导致执行器运行了错误的任务
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

代码很简单，只要新线程设置了 `shared_state.completed = true`，那任务就能顺利结束。如果没有设置，会为当前的任务克隆一份 `Waker`，这样新线程就可以使用它来唤醒当前的任务。

最后，创建一个 API 用于构建定时器和启动计时线程：

```rust
impl TimerFuture {
    /// 创建一个新的`TimerFuture`，在指定的时间结束后，该`Future`可以完成
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        // 创建新线程
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            // 睡眠指定时间实现计时功能
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            // 通知执行器定时器已经完成，可以继续`poll`对应的`Future`了
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```

对比之前的 `Delay`（用 `wake_by_ref()` 立刻通知，本质是忙轮询），`TimerFuture` 实现了真正的事件驱动——只有当计时线程 sleep 结束后，才会调用 `waker.wake()` 通知执行器。在等待期间，CPU 可以去做别的事情。

有了事件驱动的 Future，执行器也需要相应升级：不再盲目遍历所有任务，而是被动等待被唤醒的任务到来。

#### V2：基于消息通道的执行器

V1 用 `VecDeque` 主动遍历所有任务，V2 改用消息通道（channel）：执行器阻塞在接收端等待，只有被 `wake()` 唤醒的任务才会被送入通道、被 poll。

```rust
/// 任务执行器，负责从通道中接收任务然后执行
struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

/// `Spawner`负责创建新的`Future`然后将它发送到任务通道中
#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

/// 一个Future，它可以调度自己(将自己放入任务通道中)，然后等待执行器去`poll`
struct Task {
    /// 进行中的Future，在未来的某个时间点会被完成
    ///
    /// 按理来说`Mutex`在这里是多余的，因为我们只有一个线程来执行任务。但是由于
    /// Rust并不聪明，它无法知道`Future`只会在一个线程内被修改，并不会被跨线程修改。因此
    /// 我们需要使用`Mutex`来满足这个笨笨的编译器对线程安全的执着。
    ///
    /// 如果是生产级的执行器实现，不会使用`Mutex`，因为会带来性能上的开销，取而代之的是使用`UnsafeCell`
    future: Mutex<Option<BoxFuture<'static, ()>>>,

    /// 可以将该任务自身放回到任务通道中，等待执行器的poll
    task_sender: SyncSender<Arc<Task>>,
}

fn new_executor_and_spawner() -> (Executor, Spawner) {
    // 任务通道允许的最大缓冲数(任务队列的最大长度)
    // 当前的实现仅仅是为了简单，在实际的执行中，并不会这么使用
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor { ready_queue }, Spawner { task_sender })
}
```

下面再来添加一个方法用于生成 Future，然后将它放入任务通道中：

```rust
impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone(),
        });
        self.task_sender.send(task).expect("任务队列已满");
    }
}
```

接下来是关键：如何让任务在被 `wake()` 时把自己送回通道？答案是为 `Task` 实现 `ArcWake` 特征：

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // 通过发送任务到任务管道的方式来实现`wake`，这样`wake`后，任务就能被执行器`poll`
        let cloned = arc_self.clone();
        arc_self
            .task_sender
            .send(cloned)
            .expect("任务队列已满");
    }
}
```

当任务实现了 `ArcWake` 特征后，它就变成了 `Waker`，在调用 `wake()` 对其唤醒后会将任务复制一份所有权（`Arc`），然后将其发送到任务通道中。最后我们的执行器将从通道中获取任务，然后进行 `poll` 执行：

```rust
impl Executor {
    fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            // 获取一个future，若它还没有完成(仍然是Some，不是None)，则对它进行一次poll并尝试完成它
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                // 基于任务自身创建一个 `LocalWaker`
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&*waker);
                // `BoxFuture<T>`是`Pin<Box<dyn Future<Output = T> + Send + 'static>>`的类型别名
                // 通过调用`as_mut`方法，可以将上面的类型转换成`Pin<&mut dyn Future + Send + 'static>`
                if future.as_mut().poll(context).is_pending() {
                    // Future还没执行完，因此将它放回任务中，等待下次被poll
                    *future_slot = Some(future);
                }
            }
        }
    }
}
```

对比 V1 和 V2 的核心区别：

| | V1（忙轮询） | V2（按需唤醒） |
| --- | --- | --- |
| 数据结构 | `VecDeque<Task>` | `channel::Sender/Receiver` |
| 调度方式 | 不断 `pop_front` → `poll` → 没完成就 `push_back` | 阻塞在 `recv()`，只有被 `wake()` 送回通道的任务才会被 poll |
| CPU 开销 | 大量无效 poll，CPU 空转 | 无任务时线程休眠，零开销等待 |
| 核心区别 | 执行器主动轮询所有任务 | 任务主动通知执行器"我准备好了" |

#### 运行定时器 Future

下面再来写一段代码使用该执行器去运行之前的定时器 Future：

```rust
fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    // 生成一个任务
    spawner.spawn(async {
        println!("howdy!");
        // 创建定时器Future，并等待它完成
        TimerFuture::new(Duration::new(2, 0)).await;
        println!("done!");
    });

    // drop掉任务，这样执行器就知道任务已经完成，不会再有新的任务进来
    drop(spawner);

    // 运行执行器直到任务队列为空
    // 任务运行后，会先打印`howdy!`, 暂停2秒，接着打印 `done!`
    executor.run();
}
```

#### 整体运行流程总结

以上面的 `main` 函数为例，把 `Spawner`、`Executor`、`Task`、`TimerFuture`、`ArcWake` 这些组件串起来，完整的执行流程如下：

**第一阶段：初始化**

1. `new_executor_and_spawner()` 创建一个 `sync_channel`，`Executor` 持有接收端（`Receiver`），`Spawner` 持有发送端（`SyncSender`）。

**第二阶段：提交任务**

2. `spawner.spawn(async { ... })` 将 async 块包装成 `BoxFuture`，再封装进 `Task`（Task 内部也持有一份 channel 发送端），然后通过 `task_sender.send(task)` 将任务发送到通道中。
3. `drop(spawner)` 销毁 Spawner，此时只剩 Task 内部还持有发送端。这保证了：当所有 Task 都执行完毕后，通道的所有发送端都会被 drop，`recv()` 会返回 `Err`，执行器循环自然退出。

**第三阶段：首次 poll**

4. `executor.run()` 启动循环，从通道 `recv()` 取出 Task。
5. 执行器基于 Task 的 `ArcWake` 实现创建一个 `Waker`，再构造 `Context`，然后对 Future 进行第一次 `poll`。
6. async 块开始执行，打印 `"howdy!"`。
7. 遇到 `TimerFuture::new(Duration::new(2, 0)).await`：
   - `TimerFuture::new()` 创建 `SharedState`（`completed: false, waker: None`），并 spawn 一个新线程，该线程开始 sleep 2 秒。
   - `.await` 触发 `TimerFuture::poll()`，检查 `shared_state.completed` → `false`。
   - 将当前 `Waker` 克隆后存入 `shared_state.waker`，返回 `Poll::Pending`。
8. async 块也返回 `Poll::Pending`，执行器将 Future 放回 Task 中。
9. 执行器循环继续调用 `recv()`，此时通道为空，线程阻塞等待。

**第四阶段：唤醒与再次 poll**

10. 2 秒后，计时线程醒来，获取锁，设置 `shared_state.completed = true`。
11. 调用 `waker.take()` 取出之前存储的 `Waker`，执行 `waker.wake()`。
12. `wake()` 触发 Task 的 `ArcWake::wake_by_ref` 实现 → 将 Task 自身（`Arc<Task>`）重新发送到通道中。
13. 执行器的 `recv()` 收到任务，解除阻塞，对 Future 进行第二次 `poll`。
14. async 块从上次 `.await` 的位置恢复执行，`TimerFuture::poll()` 检查 `shared_state.completed` → `true`，返回 `Poll::Ready(())`。
15. async 块继续往下执行，打印 `"done!"`，整个 async 块返回 `Poll::Ready(())`。

**第五阶段：退出**

16. Future 已完成，执行器不会将其放回 Task。此时 Task 被 drop，其内部持有的 channel 发送端也随之 drop。
17. 通道所有发送端均已关闭，`recv()` 返回 `Err`，`while let` 循环退出，程序结束。

核心机制可以归纳为三个字：等、通知、跑。Future 在无法完成时注册 Waker 然后让出控制权（等），外部条件就绪时通过 Waker 将 Task 重新送入通道（通知），执行器从通道取出 Task 再次 poll 推动执行（跑）。这就是 Rust 异步运行时的本质——基于 Waker 的按需唤醒机制，避免了忙轮询的 CPU 浪费。

用一张图来概括：

![](images/img_02.png)

### 执行器和系统 IO

前面的 `TimerFuture` 用了一个专门的线程来做计时和唤醒。但在真实的网络场景中，不可能为每个 socket 都创建一个线程去轮询数据是否就绪——那样性能太低了。

回顾之前的 `SocketRead` 例子中 `set_readable_callback(wake)` 是怎么工作的？现实中，这往往是通过操作系统提供的 IO 多路复用机制来完成（Linux 的 epoll、macOS 的 kqueue、Windows 的 IOCP），可以实现一个线程同时阻塞地去等待多个异步 IO 事件，一旦某个事件完成就立即退出阻塞并返回数据。

这样，我们只需要一个执行器线程，它会接收 IO 事件并将其分发到对应的 `Waker` 中，接着后者会唤醒相关的任务，最终通过执行器 `poll` 后，任务可以顺利地继续执行，这种 IO 读取流程可以不停的循环，直到 socket 关闭。

### 本章总结

回顾整个演进过程：

**1. async fn 的本质**

`async fn` 并不会立即执行，它只是返回一个实现了 `Future` trait 的状态机。调用 `my_async_fn()` 相当于创建了一个"待执行的计划"，只有被 `.await` 或执行器 `poll` 时才会真正推进。

**2. 编译器做了什么**

编译器将 `async fn` 中每个 `.await` 点作为分界，把函数体拆分为一个枚举状态机（如 `State0 → State1 → Terminated`）。每次 `poll` 时通过 `match` 推进到下一个状态，这就是 async/await 的零成本抽象——没有堆分配，没有动态调度，只是一个普通的 `enum` + `loop` + `match`。

**3. 四个核心组件的协作**

![](images/img_03.png)

- Future：状态机，每次 `poll` 推进一步，未完成时注册 `Waker` 后返回 `Pending`
- Waker：Future 和 Executor 之间的桥梁，外部事件通过它通知执行器
- Executor：驱动循环，从通道接收就绪任务并 `poll`
- 外部事件源：真正的异步来源（定时器、IO、网络等），负责在条件就绪时调用 `wake()`

**4. 与真实 Tokio 的对比**

我们的 mini 执行器已经具备了核心骨架，真实的 Tokio 在此基础上增加了：

- 多线程调度器：work-stealing 算法，多个工作线程从共享队列中窃取任务
- IO 驱动：基于 epoll/kqueue/IOCP 的 IO 多路复用，替代我们手动 spawn 线程的方式
- 时间驱动：时间轮（timing wheel）管理大量定时器，而非每个定时器一个线程
- 协作式调度：通过预算（budget）机制防止单个任务长时间霸占线程

但万变不离其宗——poll + Waker + Executor 这三位一体的模式，就是 Rust 异步的全部核心。

![](images/img_04.png)

## Tokio

Rust 语言本身只提供了异步编程所需的基本特性，例如 `async`/`await` 关键字，这些特性单独使用没有任何用处，因此我们需要一个运行时来将这些特性实现的代码运行起来。目前最受欢迎的异步运行时就是 tokio。

### Tokio 不适用的场景

- 并行运行 CPU 密集型的任务

tokio 非常适合于 IO 密集型任务，这些 IO 任务的绝大多数时间都用于阻塞等待 IO 的结果。但是如果是 CPU 密集型（例如并行计算），不建议通过 tokio 创建异步任务来执行它；因为 tokio 是协作式的调度器，如果某个 CPU 密集的异步任务是通过 tokio 创建的，那理论上来说，该异步任务需要跟其它的异步任务交错执行，最终大家都得到了执行。但是 CPU 密集的任务很可能会一直霸着 CPU，此时 tokio 的调度方式决定了该任务会一直被执行，这意味着，其它的异步任务无法得到执行的机会，最终这些任务都会因为得不到资源而饿死。

> 但是可以使用 `spawn_blocking` 创建一个阻塞的线程去完成相应 CPU 密集任务，其会创建一个单独的 OS 线程，并不会被 tokio 所调度，它所执行的 CPU 密集任务也不会导致 tokio 调度的那些异步任务被饿死。

- 读取大量的文件

读取文件的瓶颈主要在于操作系统，因为 OS 没有提供异步文件读取接口，大量的并发并不会提升文件读取的并行性能，反而可能会造成不可忽视的性能损耗，因此建议使用线程（或线程池）的方式。

- 发送少量 HTTP 请求

tokio 的优势是给予你并发处理大量任务的能力，对于这种轻量级 HTTP 请求场景，tokio 除了增加你的代码复杂性，并无法带来什么额外的优势。

### 基本使用

```rust
use mini_redis::{client, Result};

#[tokio::main]
async fn main() -> Result<()> {
    // 建立与mini-redis服务器的连接
    let mut client = client::connect("127.0.0.1:6379").await?;

    // 设置 key: "hello" 和 值: "world"
    client.set("hello", "world".into()).await?;

    // 获取"key=hello"的值
    let result = client.get("hello").await?;

    println!("从服务器端获取到结果={:?}", result);

    Ok(())
}
```

`.await` 表示等待操作执行完毕；但是 `.await` 会将操作切到后台去等待，当前线程不会被阻塞，它会接着执行其它的 task。一旦之前的操作准备好可以继续执行后，它会通知执行器，然后执行器会调度它并从上次离开的点继续执行。

如果没有使用 `await`，而是按照这个异步的流程使用通知 -> 回调的方式实现，类似 Java 的 `whenComplete`，存在大量冗余模版代码。

### `#[tokio::main]` 原理

`#[tokio::main]` 宏将 `async fn main` 隐式的转换为 `fn main` 的同时还对整个异步运行时进行了初始化。例如以下代码：

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}
```

将被转成：

```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

### 创建异步任务

一个 Tokio 任务是一个异步的绿色线程，它们通过 `tokio::spawn` 进行创建，该函数会返回一个 `JoinHandle` 类型的句柄，调用者可以使用该句柄跟创建的任务进行交互。

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
       10086
    });

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

`tokio::spawn` 生成的任务必须实现 `Send` 特征，因为当这些任务在 `.await` 执行过程中发生阻塞时，Tokio 调度器会将任务在线程间移动。

一个任务要实现 `Send` 特征，那它在 `.await` 调用的过程中所持有的全部数据都必须实现 `Send` 特征。当 `.await` 调用发生阻塞时，任务会让出当前线程所有权给调度器，然后当任务准备好后，调度器会从上一次暂停的位置继续执行该任务。该流程能正确地工作，任务必须将 `.await` 之后使用的所有状态保存起来，这样才能在中断后恢复现场并继续执行。

例如以下代码可以工作：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // 语句块的使用强制了 `rc` 会在 `.await` 被调用前就被释放，
        // 因此 `rc` 并不会影响 `.await`的安全性
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` 的作用范围已经失效，因此当任务让出所有权给当前线程时，它无需作为状态被保存起来
        yield_now().await;
    });
}
```

但是下面代码就不行：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` 在 `.await` 后还被继续使用，因此它必须被作为任务的状态保存起来
        yield_now().await;

        // 事实上，注释掉下面一行代码，依然会报错
        // 原因是：是否保存，不取决于 `rc` 是否被使用，而是取决于 `.await`在调用时是否仍然处于 `rc` 的作用域中
        println!("{}", rc);

        // rc 作用域在这里结束
    });
}
```

报错：

```
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

## 异步同步共存

如何在同步代码中使用一小部分异步代码。

在 Rust 中，`main` 函数不能是异步的，而之前我们通过 `async fn main` + `#[tokio::main]` 的声明，是因为 `#[tokio::main]` 仅仅是提供语法糖，目的是让大家可以更简单、更一致的去写异步代码，它会将你写下的 `async fn main` 函数替换为：

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("Hello world");
        })
}
```

注意到上面的 `block_on` 方法，在我们自己的同步代码中，可以使用它开启一个 async/await 世界。

```rust
use tokio::runtime::Builder;
use tokio::time::{sleep, Duration};

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(1)
        .enable_all()
        .build()
        .unwrap();

    let mut handles = Vec::with_capacity(10);
    for i in 0..10 {
        handles.push(runtime.spawn(my_bg_task(i)));
    }

    // 在后台任务运行的同时做一些耗费时间的事情
    std::thread::sleep(Duration::from_millis(750));
    println!("Finished time-consuming task.");

    // 等待这些后台任务的完成
    for handle in handles {
        // `spawn` 方法返回一个 `JoinHandle`，它是一个 `Future`，因此可以通过  `block_on` 来等待它完成
        runtime.block_on(handle).unwrap();
    }
}

async fn my_bg_task(i: u64) {
    let millis = 1000 - 50 * i;
    println!("Task {} sleeping for {} ms.", i, millis);

    sleep(Duration::from_millis(millis)).await;

    println!("Task {} stopping.", i);
}
```
