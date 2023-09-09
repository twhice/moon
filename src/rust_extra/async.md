# 异步

*这里假定你已经阅读完成 [rust基础](../rust_basics/intro.md) 和 ~~我还没写~~ 的[rust多线程]()*

*异步一度被认为是一个晦涩的话题，但是我觉得：还好*

# 初探

观看以下的代码

```rust
fn main() {
    fn hello_world() {
        println!("Hello World");
    }

    async fn async_hello_world() {
        println!("Hello World");
    }

    let normal = hello_world();

    let feature = async_hello_world();
}
```

*使用async fn 声明一个异步函数*

我声明了一个普通函数`hello_world`，和它的异步版本`async_hello_world`，然后调用了它们

你可以执行这段代码，你会发现：只有一行`Hello World`出现

如果你将这段代码复制进编辑器，你可以看到`feature`的类型是`impl Feature<Output = ()>`。这不是一个具体的类型，只是编译器为了省略一些细节而显示给你的

它的意思是`一个实现了Feature，并且Feature的关联类型Output为()的类型`

通过删除`let normal = hello_world();`，再运行，你应该可以得出这样的一个结论：`async_hello_world()`并没有运行`async_hello_world`这个函数，而是返回了一个奇奇怪怪的东西

就像是一个函数指针一样，它确实代表着一个函数，但是你可以拿到代表这个函数的函数指针而不运行它，并且可以通过函数指针运行函数

不过让我们先像使用函数指针一样，真正使用这个奇奇怪怪的东西，让第二个Hello World出现

打开终端，输入以下，来添加tokio作为依赖
```bash
cargo add tokio --features=full
```

然后将代码改成这样
```rust
#[tokio::main]
async fn main() {
    fn hello_world() {
        println!("Hello World");
    }

    async fn async_hello_world() {
        println!("Hello World");
    }

    let normal = hello_world();

    let feature = async_hello_world();
    let result = feature.await;
}
```

同样运行，可以看见第二个Hello World也被打印出来了，result的类型也显示为()。可以看得出，这是`.await`的功劳，它实际执行了这个奇奇怪怪的东西，然后给出了结果

但是如果你把
```rs
#[tokio::main]
async fn main(){
}
```
改回
```rs
fn main(){
}
```
或者仅仅删掉`#[tokio::main]`

你会收获这样的一些编译错误之一

* `main` function is not allowed to be `async`

* `await` is only allowed inside `async` functions and blocks

所以说，#[tokio::main]的作用是让main可以是一个异步的函数，而让main变成异步函数的目的就是对`feaure`进行`.await`，真正执行`async_hello_world`

你可能还是一头雾水，不过，请看下文

# 异步运行时

经过上面的例子和尝试，你一定发现了异步函数得要特殊方法才能运行

也就是说，异步需要一个**异步运行时**才可以运行

`async_hello_world`这个异步函数通过`.await`才“真正执行”，而`.await`又需要放在一个异步函数才可以用。异步函数就这样层层嵌套，那么嵌套的终点，就是main函数

而main不可以是异步函数，所以才需要添加`#[tokio::main]`这个属性来修改main函数，用`.await`以外的方式执行了"main"返回的`impl Feature<Output = ...>`

这个”`.await`以外的方式“就是运行时。显而易见，await仅仅是起到了“传递”的作用，真正运行的地方只在运行时。

tokio就是一个异步运行时，并且提供了很多io上的结构/方法供给异步编程使用

还记得在多线程那节说的操作系统运行无数进程的方法吗？让每个进程运行一会，停一会

实际上，`.await`真正的作用就是，让运行时“停下来”。也就是切换到别的任务中

看看这个异步函数，它内部有两个异步块，每个块返回的`impl Feature<Output = ()>`类型的东西，都被`.await`了
```rust
async fn x() {
    async {
        let two = 1 + 1;
    }
    .await;
    async {
        let this_file = tokio::fs::read_to_string("src/main.rs").await.unwrap();
        println!("{}", this_file)
    }
    .await;
}
```
运行时运行这个异步函数时会发生什么呢？

运行时会立马执行第一个`Feature`，然后运行第二个`Feature`。但是运行第二个`Feature`时不会像第一个一样立刻得到结果，而是得等待操作系统读取文件，才能拿到`this_file`，然后打印

这时，运行时在运行到`this_file`的`.await`就会停下，等待操作系统给出数据。而这个等待的时间，是不是可以拿来干别的？等到运行时干完了别的，操作系统或许也返回了数据，当运行时在看看这个`.await`，发现已经好了，就可以继续执行下面的代码了

实际上呢，运行时也不一定执行完第一个`Feature`就马不停蹄地运行第二个`Feature`，它也可能跑去运行别的任务

这就是异步的特点，不同于多线程，异步运行时切换到一个任务不再只有“做”这一个选择，它可以明确地得知它应该转去执行别的任务，而不是空耗时间

# 对比线程

*线程也是这样的啊，我创建一个线程一样是分时间，为什么还需要额外多一个异步呢？*



看看这个函数
``` rs
fn x() {
    let h1 = std::thread::spawn(|| {
        let two = 1 + 1;
    });
    let h2 = std::thread::spawn(|| {
        let this_file = std::fs::read_to_string("src/main.rs").unwrap();
        println!("{}", this_file)
    });
    h1.join().unwrap();
    h2.join().unwrap();
}
```

线程的创建实际上时***非常昂贵***的，非常费时间，操作系统为了创建一个线程，会进行分配栈空间等大量工作，但是这些东西都不一定用得上。

对于h1，整个运算至多用上十几byte，但是操作系统为此创建了几M的栈，以及一堆其他的，然后很快又销毁栈

对于h2，操作系统的调度器运行到了h2时并不会因为是等待而马上切换到下一个任务，而是傻乎乎地等待。某种层面上讲，多线程甚至徒增了切换线程的成本

而对于这个并行的异步函数
```rs
async fn x() {
    let h1 = tokio::spawn(async {
        let two = 1 + 1;
    });

    let h2 = tokio::spawn(async {
        let this_file = tokio::fs::read_to_string("src/main.rs").await.unwrap();
        println!("{}", this_file)
    });

    h1.await.unwrap();
    h2.await.unwrap();
}

```
你会发现代码非常相似

但是，`tokio::spawn`并不会真正创建一个线程，它只是让运行时在运行到其中一个“线程”的某个.await时，运行时可以切换到另一个”线程“。如果运行时是一个多线程的就更好了，h1 h2同时运行

h1 h2的实际类型时编译器创建的匿名的结构体，就像是闭包一样，h1 h2需要用到的东西就已经被存放在了结构体里。这样的一个结构体就是一个”任务“，它仅仅占用很小的空间而且在代码编译时就可以准备好。

# Feature

查看Feature的定义，相信你可以有一个更清晰的认知


删除文档和属性后，这就是Feature特征的定义
```rs
pub trait Future {
    type Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}

```
Output是一个关联类型，也就是一个Feature最后返回的类型

poll方法由运行时调用，poll返回一个枚举

Poll::Pending表示这个异步任务还没准备好，运行时就可以立刻切换到别的任务

Poll::Ready表示这个异步任务已经准备好了，于是运行时就运行poll，取得T

运行时并不会一直调用poll等Feature准备好，而是等Feature准备好后用Context向运行时发出”我准备好了“的信息

### 那么Pin是什么？


看看这样的一个结构体：
``` rs
struct SelfRef {
    usize: usize,
    poinger: *const usize,
}

impl SelfRef {
    pub fn new(u: usize) -> Self {
        let mut s = Self {
            usize: u,
            poinger: std::ptr::null(),
        };
        s.poinger = s.usize as *const usize;
        s
    }

    pub fn get(&self) -> usize {
        unsafe { *self.poinger }
    }
}
```
这是一段典型的ub代码。这个结构体内部有一个指针，指向了它自身的另一个元素，这是一个自引用结构

它的get方法假设了pointer的地址就是usize的地址，但是实际上因为new方法最后移动了s，pointer在new结束后可能就不指向usize了

这个结构体的每次移动，都可能把usize带到一个新的内存地址，但是pointer仍旧指向原来的地址

一旦get被调用，后果不可预料

*你可能会觉得这个例子，有些刻意。但是实际上这就是简化一个场景*

不同于闭包会一次性执行结束，一个Feature可能会执行好几次，它自己会保存状态。如果一个Feature是自引用的，然后它因为任何原因被移动，使用它就会是ub

*只要有ub的可能，就要防患于未然*

为了应付自引用，就有了Pin，它的特点是无法被移动，试图移动一个Pin会触发编译错误，使得以外的移动不会发生

# 异步编程