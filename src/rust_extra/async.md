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

所以说，`#[tokio::main]`的作用是让main可以是一个异步的函数，而让main变成异步函数的目的就是对`feaure`进行`.await`，真正执行`async_hello_world`

你可能还是一头雾水，不过，请看下文

# 异步运行时

经过上面的例子和尝试，你一定发现了异步函数得要特殊方法才能运行

也就是说，异步需要一个**异步运行时**才可以运行

`async_hello_world`这个异步函数通过`.await`才“真正执行”，而`.await`又需要放在一个异步函数才可以用。异步函数就这样层层嵌套，那么嵌套的终点，就是main函数

而main不可以是异步函数，所以才需要添加`#[tokio::main]`这个属性来修改main函数，用`.await`以外的方式执行了"main"返回的`impl Feature<Output = ...>`

这个”`.await`以外的方式“就是运行时。显而易见，await仅仅是起到了“传递”的作用，真正运行的地方只在运行时。

tokio提供了异步运行时，和很多实用的结构/方法供给异步编程使用

*标准库没有异步运行时*

还记得在多线程那节说的操作系统运行无数进程的方法吗？让每个进程运行一会，停一会

实际上，`.await`真正的作用就是，让运行时可以“停下这个任务”，切换到别的任务中

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

运行时在运行到`this_file`的`.await`时就会停下，等待操作系统给出数据。这个等待的时间，运行时可以转移去运行别的任务。等到运行时干完了别的，操作系统或许也返回了数据，当运行时在查看这个`.await`，发现已经好了，就可以继续执行下面的代码了，打印`this_file`了

这就是异步的特点，不同于多线程，异步运行时切换到一个任务不再只有“做”这一个选择，它可以明确地得知它应该转去执行别的任务，而不是空耗时间在等待上

# 对比线程

*线程也是这样的啊，我创建一个线程一样是分时间，为什么还需要异步呢？*

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

线程的创建实际上时***非常昂贵***的，非常费时间，操作系统为了创建一个线程，会进行大量准备工作，为了切换线程还有大量运行时开销，但是这些东西都不一定用得上。

对于h1，整个运算至多用上十几byte，但是操作系统为此创建了几M的栈，以及一堆其他的，然后很快又销毁栈

对于h2，操作系统的调度器运行到了h2时，并不会因为h2在等待而马上切换到下一个任务，而是傻乎乎地等待。某种层面上讲，多线程甚至徒增了切换线程的成本

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
你会发现代码和上一个非常相似

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

运行时并不会一直调用poll等Feature准备好，而是等Feature准备好后通过Context向运行时发出”我准备好了“的信息

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

为了应付自引用，就有了Pin，它的特点是无法被移动，试图移动一个Pin会触发编译错误，使得意外的移动不会发生

# 异步编程

好了，是时候开始真正关心的问题了：*进行异步编程*

*概念是为实践服务，不是么*

## 配置依赖

rust std没有一个异步运行时，只提供基本的`Feature`之类的，想要写代码这也是远远不够的

所以将tokio添加到依赖里

feature我这里是`full`，简单省事，如果你了解各个feature是什么，可以自行配置

~~我也没了解tokio的各个feature~~

```toml
# Carog.toml
[dependencies]
tokio = { version = "1.32.0", features = ["full"] }
```

## 多线程

你得意识到两个问题：

1. 异步是多线程还是单线程，得看运行时的实现。`#[tokio::main]`默认是多线程的异步运行时

2. 要多线程执行异步任务，你仍然得像普通多线程一样spawn一个”线程“出来

## 例子

写一个读取文件夹下全部文件内容的异步函数

```rust
# use std::path::Path;

# #[tokio::main]
# async fn main() -> std::io::Result<()> {
#     dbg!(read_all_file(".").await?);
#     Ok(())
# }
# 
async fn read_all_file<P: AsRef<Path>>(p: P) -> std::io::Result<Vec<String>> {
    use tokio::fs;

    let mut dir = fs::read_dir(p).await?;

    let mut result = vec![];

    while let Some(entry) = dir.next_entry().await? {
        if entry.file_type().await.is_ok_and(|t| t.is_file()) {
            result.push(fs::read_to_string(entry.path()).await?)
        }
    }

    Ok(result)
}
```

这个函数可圈可点的地方在哪呢？

* 文件(IO)操作用的都是`tokio::fs`而非`std::fs` **否则不会有任何性能提升**

看起来ok对吧，但是实际上这**完全是单线程的**，和同步的版本性能差别不大

为什么？毕竟你就是按照类似同步的方法写的啊，为什么觉得会自动获得多线程啊？

它的多线程版本大概是这样的

```rust
# use std::path::Path;

# #[tokio::main]
# async fn main() -> std::io::Result<()> {
#     dbg!(read_all_file(".").await?);
#     Ok(())
# }
# 
async fn read_all_file<P: AsRef<Path>>(p: P) -> std::io::Result<Vec<String>> {
    use tokio::fs;

    let mut dir = fs::read_dir(p).await?;

    let mut result = vec![];
    let mut handles = vec![];

    while let Some(entry) = dir.next_entry().await? {
        handles.push(tokio::spawn(async move {
            if entry.file_type().await.is_ok_and(|t| t.is_file()) {
                Some(fs::read_to_string(entry.path()).await)
            } else {
                None
            }
        }));
    }

    for handle in handles {
        let Ok(Some(r)) = handle.await else {
            continue;
        };
        result.push(r?);
        
    }

    Ok(result)
}
```
就像是普通的多线程一样，创建一个集合存储`JoinHandle`，创建一堆线程然后一个个join

不过对于`std::thread::spawn`，这样创建一堆线程的行为十分不可取，因为线程创建销毁的成本巨大，这里的创建/销毁又是很频繁的，线程池可能是更好的选择。

但是对于此：`tokio::spawn`来说，开销特别小。你可以这样理解：`tokio::spawn`仅仅为任务做准备，然后委托给运行时运行，而`std::thread::spawn`会为了一个准备运行这个任务所需要的运行时

**但是，如果你的IO操作使用的不是异步的，就不会有一个很好的性能，因为那样tokio没有进行任务切换的机会**

也有一些细节需要注意：

* 使用了`Option::None`表示不是文件这种情况

* `handle.await`  的 `Result<T,E>` 的 E，指的是异步任务发生panic之类的

* 对文件读取操作的错误，传递出去了

不过，就像用map这样的方法替代特定的if let之类的一样，这其实有个更好的版本
``` rust
# use std::path::Path;

# #[tokio::main]
# async fn main() -> std::io::Result<()> {
#     dbg!(read_all_file(".").await?);
#     Ok(())
# }
# 
async fn read_all_file<P: AsRef<Path>>(p: P) -> std::io::Result<Vec<String>> {
    use tokio::fs;

    let mut dir = fs::read_dir(p).await?;

    let mut result = vec![];

    let mut set = tokio::task::JoinSet::new();

    while let Some(entry) = dir.next_entry().await? {
        set.spawn(async move {
            if entry.file_type().await.is_ok_and(|t| t.is_file()) {
                Some(fs::read_to_string(entry.path()).await)
            } else {
                None
            }
        });
    }

    while let Some(r) = set.join_next().await {
        let Ok(Some(r)) = r else {
            continue;
        };
        result.push(r?)
    }

    Ok(result)
}
```
这里使用`JoinSet`替代了集合，更加直观方便

 