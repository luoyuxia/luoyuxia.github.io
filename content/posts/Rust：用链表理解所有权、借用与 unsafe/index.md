---
title: "Rust：用链表理解所有权、借用与 unsafe"
date: 2026-04-12T15:36:30+08:00
draft: false
tags: ["Rust"]
categories: ["Rust"]
summary: "本文通过五版链表实现，系统演示Rust所有权、借用、生命周期、Rc/RefCell共享与内部可变性，以及unsafe裸指针等核心机制的演进与权衡。"
ShowToc: true
TocOpen: false
---

本文通过实现四种不同的链表，逐步深入 Rust 的核心机制。每一版链表都会暴露新的问题，驱动我们学习新的语言特性：

| 版本 | 数据结构 | 核心知识点 |
|------|----------|-----------|
| v1 不太优秀的单向链表 | `Box` + 自定义 `enum Link` | 内存布局、所有权转移、`mem::replace` |
| v2 还可以的单向链表 | `Option<Box<Node>>` + 泛型 | `take()`、`as_ref()`、生命周期、三种迭代器 |
| v3 持久化单向链表 | `Rc` 共享所有权 | 引用计数、不可变共享、`Rc::try_unwrap` |
| v4 双端队列 | `Rc<RefCell<Node>>` | 内部可变性、`borrow_mut()`、`Ref::map` 的局限 |
| v5 unsafe 队列 | 裸指针 `*mut Node` | `Box::into_raw`、`Box::from_raw`、Miri、UB |

## v1：不太优秀的单向链表（栈）

### 基本布局

最直觉的实现——用枚举递归定义链表：

```rust
pub enum List {
    Empty,
    Elem(i32, List),
}
```

编译器直接报错：

```
1 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
3 |     Elem(i32, List),
  |               ---- recursive without indirection
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to make `List` representable
```

`List` 是一个递归类型，编译器无法在编译期确定它的大小。为什么确定大小这么重要？因为 Rust 的函数调用依赖它——栈帧的布局（每个局部变量的偏移量、函数栈帧的总大小）在编译期就要确定，运行时不能动态调整：

```rust
fn foo() {
    let x: List = List::Empty;  // 编译器需要知道：栈帧给 x 留多少字节？
}
```

而编译器计算 `List` 大小时会陷入无限递归：

- `List` 的大小 = max(`Empty`, `Elem`) = `Elem` 的大小
- `Elem` 的大小 = `i32`(4 字节) + `List` 的大小
- `List` 的大小 = 4 + `List` 的大小 = 4 + 4 + `List` 的大小 = ...

算不出一个确定的数字，所以拒绝编译。解决办法是加一层间接引用 `Box`，让递归部分变成一个固定大小的堆指针：

```rust
pub enum List {
    Empty,                 // 0 字节
    Elem(i32, Box<List>),  // 4 字节 + 8 字节(指针) = 12 字节（再加对齐）
}
// List 大小 = max(0, 12) = 确定了！
```

能编译了，但内存布局有两个问题：

问题 1：空节点浪费空间。由于 `enum` 的内存对齐，`Empty` 变体也要占用和 `Elem` 一样大的空间（至少要存下最大变体的大小）。

问题 2：首节点在栈上，其余在堆上。这导致分割/合并链表时需要在栈和堆之间搬运数据：

```
布局（当前）：
[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

分割 C 后：
[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)   ← C 从堆上搬到了栈上，产生额外拷贝
```

更好的布局是：所有节点都在堆上，链表本身只持有一个指针，空尾用 null 表示而非一个 `Empty` 节点：

```
布局（理想）：
[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

分割 C 后：
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)   ← 只需改指针，无需拷贝
```

如下图所示：

![](images/img_01.png)

为此，我们将链表拆分为三个类型：

```rust
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

`List` 只是一个指针大小的栈上结构，所有 `Node` 都通过 `Box` 分配在堆上，空尾用 `Link::Empty` 表示（不占额外空间，因为编译器会将 `Box` 的 null 指针优化为 `Empty` 变体）。

### Push

创建一个新节点，让它的 `next` 指向当前 `head`，然后更新 `head` 指向新节点：

```rust
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
        self.head = Link::More(Box::new(new_node));
    }
}
```

编译报错：

```
error[E0507]: cannot move out of `self.head` which is behind a mutable reference
  --> src/first.rs:23:19
   |
23 |             next: self.head,
   |                   ^^^^^^^^^ move occurs because `self.head` has type `Link`,
   |                             which does not implement the `C
```

`self` 是一个 `&mut` 借用，我们试图将 `self.head` 的所有权移动给 `next`，这会让 `self` 处于一个不完整的状态（`head` 被拿走了但还没放回新值），Rust 不允许这样做

不太优的方案：clone

```rust
#[derive(Clone)]
enum Link { ... }

#[derive(Clone)]
struct Node { ... }

pub fn push(&mut self, elem: i32) {
    let new_node = Node {
        elem: elem,
        next: self.head.clone(), // 深拷贝整条链表，O(n) 开销
    };
    self.head = Link::More(Box::new(new_node));
}
```

能编译，但 `clone()` 会深拷贝从 `head` 开始的整条链表，完全不可接受。

更优的方案：`mem::replace`

我们需要的是：从 `self.head` 中"偷"出值，同时放入一个临时占位值，保证 `self` 始终处于合法状态：

`mem::replace` 原理如下图所示：

![](images/img_02.png)

```rust
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: std::mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

`mem::replace(&mut self.head, Link::Empty)` 做了两件事：把 `self.head` 的值取出来返回，同时把 `Link::Empty` 放进去。这样 `self` 在整个过程中始终是完整的。

### Pop

```rust
pub fn pop(&mut self) -> Option<i32> {
    match std::mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

同样使用 `mem::replace` 先把 `head` 偷出来，再根据情况处理。

## v2：还可以的单向链表

v1 有两个不优雅的地方：

1. 每次操作都要用 `mem::replace`，过于 hack

2. 额外定义了一个 `Link` 枚举，其实 Rust 内置的 `Option` 完全能胜任

### 用 Option 替代 Link

```rust
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;
// 等价于之前的：
// enum Link { Empty, More(Box<Node>) }

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

`Option` 自带 `take()` 方法，功能等同于 `mem::replace(&mut self.head, None)`，代码立刻清爽了：

```rust
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(), // 取出 head，原位留下 None
        });
        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}
```

### Peek

返回链表表头元素的引用：

```rust
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```

编译报错：

```
error[E0515]: cannot return reference to local data `node.elem`
error[E0507]: cannot move out of borrowed content
```

问题在于 `map` 会拿走 `self.head` 的所有权（`Option<T>` 的 `map` 消费 `self`），闭包内的 `node` 是一个局部变量，函数返回后就会被销毁，我们不能返回它的引用。

解决办法：先用 `as_ref()` 将 `Option<Box<Node>>` 转换成 `Option<&Box<Node>>`，这样 `map` 操作的就是引用而非所有权：

```rust
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

`as_ref()` / `as_mut()` 是 Option 编程中极其常用的方法：

```rust
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;   // Option<T> → Option<&T>
    pub fn as_mut(&mut self) -> Option<&mut T>; // Option<T> → Option<&mut T>
}
```

### 迭代器

集合类型应该实现 3 种迭代器：

| 迭代器 | 产出类型 | 语义 |
| ------ | -------- | ---- |
| `IntoIter` | `T` | 消费集合，转移所有权 |
| `Iter` | `&T` | 不可变借用遍历 |
| `IterMut` | `&mut T` | 可变借用遍历 |

它们都实现同一个 trait：

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

#### IntoIter

最简单，直接消费链表，每次 `next` 就是一次 `pop`：

```rust
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}
```

不涉及引用，不涉及生命周期，最省心。

#### Iter

持有一个指向当前节点的引用，每次 `next` 返回当前元素的引用并前进到下一个节点：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

这里有几个生命周期的要点：

- `Iter<'a, T>` 需要声明生命周期 `'a`，因为它内部持有引用
- `Iterator` 的 impl 也要带 `'a`，因为关联类型 `Item = &'a T` 需要它
- `iter(&self)` 方法不需要显式标注生命周期（生命周期消除规则自动推导）
- `as_deref()` 将 `Option<&Box<Node>>` 转换成 `Option<&Node>`，穿透 Box 直接拿到内部引用

`map` 在这里能正常工作，是因为 `Option<&T>`（不可变引用）实现了 `Copy`，`map` 拿到的只是引用的副本，不会转移所有权。

#### IterMut

照着 `Iter` 改成可变版本：

```rust
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

注意这里用了 `self.next.take()` 而不是 `self.next.map(...)`。原因在于 `&mut T`（可变引用）不可 Copy——同一时刻只能有一个可变引用存在。如果用 `map`，它会试图移动 `self.next` 的值，但 `self` 是借用的，不允许移动。`take()` 通过"取出并留下 None"来获得所有权，完美解决。

## v3：持久化单向链表

前面的链表都是单所有权的。在实际使用中，共享所有权更常见：

![](images/img_03.png)

```
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

节点 B 被三个链表共享，这带来两个问题：

1. `Box` 是独占所有权的，无法让多个链表指向同一个节点

2. 当 `list2` 被 drop 时，B 可能还被 `list1` 和 `list3` 引用，不能直接释放

解决方案：`Rc`（Reference Count 引用计数）。`Rc` 允许多个所有者共享同一份数据，当最后一个 `Rc` 被 drop 时数据才会被释放。不过 `Rc` 的数据是不可变的（如果需要可变，可以配合 `RefCell`）。

### 数据布局

```rust
use std::rc::Rc;

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

### 基本操作

链表是不可变的，所以没有 `push`/`pop`。取而代之的是返回新链表的 `prepend` 和 `tail`：

```rust
// 在头部追加元素，返回新链表（原链表不受影响）
pub fn prepend(&self, elem: T) -> List<T> {
    List {
        head: Some(Rc::new(Node {
            elem: elem,
            next: self.head.clone(), // Rc 的 clone 只增加引用计数，O(1)
        })),
    }
}

// 返回去掉头节点的新链表
pub fn tail(&self) -> List<T> {
    List {
        head: self.head.as_ref().and_then(|node| node.next.clone()),
    }
}

// 返回头节点元素的引用
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

### 自定义 Drop

链表很长时，默认的递归 drop 可能栈溢出。手动实现迭代式 drop：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break; // 还有其他引用，不能释放，停止
            }
        }
    }
}
```

`Rc::try_unwrap` 检查当前 `Rc` 是否只剩一个强引用：是则返回内部值（可以安全释放），否则返回错误（说明还有其他链表在共享这个节点）。

## v4：不太优秀的双端队列

双向链表意味着每个节点同时指向前一个和后一个节点。节点被多方持有（前驱、后继、链表头尾），需要共享所有权；同时还要能修改 `prev`/`next` 指针，需要内部可变性。这就是 `Rc<RefCell<Node>>` 的经典应用场景。

双端队列结构：

![](images/img_04.png)

### 数据布局

```rust
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

### 基本操作

```rust
impl<T> Node<T> {
    fn new(elem: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            elem: elem,
            prev: None,
            next: None,
        }))
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

#### push_front

```rust
pub fn push_front(&mut self, elem: T) {
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            old_head.borrow_mut().prev = Some(new_head.clone());
            new_head.borrow_mut().next = Some(old_head);
            self.head = Some(new_head);
        }
        None => {
            self.tail = Some(new_head.clone());
            self.head = Some(new_head);
        }
    }
}
```

#### pop_front

```rust
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        // Rc::try_unwrap 确认只剩一个引用后取出内部值
        // .into_inner() 消费 RefCell，取出 Node
        Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
    })
}
```

这里不能直接 `old_head.into_inner()`，因为 `Rc<T>` 只提供不可变引用，不允许直接消费内部值。必须先用 `Rc::try_unwrap` 确认只剩一个强引用，拿到 `RefCell<Node>`，再用 `into_inner()` 取出 `Node`。

### 迭代器

#### IntoIter

消费式迭代器比较简单，正向用 `pop_front`，反向用 `pop_back`：

```rust
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        self.0.pop_front()
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

#### Iter：此路不通

尝试实现借用迭代器：

```rust
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;

    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

编译报错：

```
error[E0521]: borrowed data escapes outside of closure
    |
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here
```

问题在于 `RefCell::borrow()` 返回的 `Ref` 的生命周期与 `RefCell` 绑定，而不是与数据本身绑定。我们需要在使用 `node_ref`（当前节点的 `Ref`）的同时，从它内部借用下一个节点——这要求 `node_ref` 活得比闭包更久，但它的所有权马上就要被 `Ref::map` 消费掉。

尝试用 `Ref::map_split` 拆分也无济于事，根本矛盾是：`Ref` 不允许被拆分成跨越多个 `RefCell` 的引用。

> `Rc<RefCell>` 双向链表的借用迭代器在安全 Rust 中基本无法实现。这是内部可变性的固有局限——`RefCell` 的运行时借用检查无法像编译期借用检查那样灵活地拆分引用。

## v5：不错的 unsafe 队列

`Rc<RefCell>` 的方案过于复杂且有诸多限制。对于需要可变性的链表场景，裸指针 + `unsafe` 反而是更务实的选择。

Unsafe 裸指针内存管理:

![](images/img_05.png)

### 第一版：混用 Box 和裸指针

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // 裸指针，指向尾节点，方便尾部追加
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```rust
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    if !self.tail.is_null() {
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}

pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

能跑，但混用 Box 和裸指针会导致未定义行为（UB：Undefined Behavior）。原因在于 Rust 的别名模型（Stacked Borrows）：`Box<T>` 声称自己是数据的唯一所有者，但我们同时通过裸指针修改同一块内存，破坏了这个契约。

```rust
// 类似这样的问题：
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1; // Miri 报错：UB！

    println!("{}", data);
}
```

Rust 认为 `Box` 是数据的唯一修改者。但裸指针也在修改同一块内存，编译器的优化（例如基于独占引用的缓存优化）可能导致非预期行为。

原则：一旦开始使用裸指针，就应该全程使用裸指针，避免与 `Box` 的所有权模型冲突。

### 第二版：纯裸指针

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // 全部用裸指针

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

#### Push

```rust
pub fn push(&mut self, elem: T) {
    unsafe {
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```

`Box::into_raw` 将 `Box` 转换为裸指针并放弃所有权（不会自动释放内存）。从这一刻起，内存管理完全由我们手动负责。

#### Pop

```rust
pub fn pop(&mut self) -> Option<T> {
    unsafe {
        if self.head.is_null() {
            None
        } else {
            let head = Box::from_raw(self.head); // 重新接管所有权
            self.head = head.next;

            if self.head.is_null() {
                self.tail = ptr::null_mut();
            }

            Some(head.elem)
        }
    }
}
```

`Box::from_raw` 将裸指针重新包装为 `Box`，恢复自动内存管理。当 `head` 离开作用域时，`Box` 会自动释放内存。

#### Peek

```rust
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

裸指针的 `as_ref()` / `as_mut()` 将 `*mut T` 转换为 `Option<&T>` / `Option<&mut T>`，null 指针会变成 `None`。

#### 迭代器

三种迭代器的实现与 v2 的安全版本结构一致，只是内部用裸指针操作：

```rust
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

```rust
impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe { Iter { next: self.head.as_ref() } }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe { IterMut { next: self.head.as_mut() } }
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
```

总结：用 c 的方式写 rust 真爽

## 总结

通过五个版本的链表实现，我们逐步触碰了 Rust 的核心机制：

| 遇到的问题 | 学到的知识 |
| ---------- | ---------- |
| 递归类型无限大 | `Box` 堆分配，间接引用 |
| 不能从 `&mut self` 中移走值 | `mem::replace`、`Option::take` |
| 返回内部引用时所有权冲突 | `as_ref()` / `as_deref()`，引用而非所有权 |
| 不可变引用需要标注存活范围 | 生命周期 `'a`、生命周期消除规则 |
| `&mut T` 不可 Copy | `take()` 获取所有权 vs `map` 拷贝引用 |
| 多个所有者共享数据 | `Rc` 引用计数 |
| 共享的同时需要修改 | `RefCell` 内部可变性 |
| `RefCell` 的 `Ref` 无法跨节点拆分 | 安全 Rust 的固有局限 |
| `Box` 和裸指针混用导致 UB | Stacked Borrows 别名模型、Miri 检测 |
| 需要完全手动管理内存 | `Box::into_raw` / `Box::from_raw`、裸指针 |

一句话总结：链表是 Rust 所有权系统的天然对手。正是因为链表的节点互相引用、共享、可变，才把 Rust 的每一道安全防线都触发了一遍。理解了链表，就理解了 Rust 为何如此设计。
