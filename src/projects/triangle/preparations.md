# 准备工作

进行图形开发，需要的准备工作是极多的，因为很多相关的配置需要进行，相关的知识需要学习

# 窗口系统


图形程序是显示在一个窗口里的，那么相应的，就有

* 负责管理窗口的程序（窗口管理器）

* 一些相关概念

> 不过需要注意的是，图形开发的很多东西**不是"跨窗口管理器"的**，使用不支持的东西是会导致运行时的*奇怪的问题*或者**直接崩溃**，这些都有在文档照中标注出

## 窗口系统&疑难杂症

这是在平台之上再次细分出的概念，有的操作系统 ~~比如linux~~ 上有很多个窗口系统供给选择

> **得益于winit，这些混乱的差异我们不需要管，我们可以用一个方法做一种事而不是为了每一种窗口系统编写不同的代码。最多只需要在使用功能时多注意目标的窗口系统支不支持这个功能就可以了就可以了**

### Windows的win32

万幸，windows是统一的。windwos的窗口管理器(应该)是一套叫做`win32`的api，我想不会有很多奇怪问题 ~~反正我没碰见过，真碰见了也应该不是我可以解决的~~

### Linux的Xorg

<div class="warning">

> ***这部分介绍节写的很差，根本不用看，不过你也可以了解一下***
>
> Xorg出问题基本都是[驱动问题](#驱动问题)

</div>


> X11是X Windows System的第11个版本，而Xorg是它的实现，这也是如今最盛行的版本。*Xorg只是一个软件，并不是linux的组成部分！*

这是一个诞生于上世纪(1984)的古老系统，四处漏风但是缝缝补补接着用

过去的硬件贫乏且羸弱，个人电脑不盛行，所以Xorg实际上分为三个部分:X Server（X服务器），X Client(X客户端)，和Windows Manager（简称WM，翻译叫做窗口管理器）

> **X Server：**
>
> X Server负责和驱动交互，监视显示器，键盘鼠标，相应X Client的请求。它也是渲染实际进行的地方

> **X Client**
>
> 也叫“X应用程序”，是实际上的处理逻辑的地方，可以接受设备的事件来进行处理。它本身没有绘制能力，只能向X Server发送绘制请求和绘图数据。

> **Window Manager**
>
> 也有叫合成器（Compositor），是一个特殊的X Client程序
>
> 多个X Client向X Server发送绘制请求时，各X Client程序并不知道彼此的存在，绘制图形出现冲突是很有可能的
>
> 这就需要一个管理者进行统一协调，即Window Manager，它掌管各X Client的Window（窗口）外观，位置，尺寸，重叠等等
>
> 它可以直接和本机上的X Server连接，也可以通过网络

基本上配置好了桌面环境的linux都支持/使用的是Xorg，但是你使用的WM也很有可能用的不是Xorg,而是Wayland
> WM也是Wayland的一个概念

X在每个部分都有很多实现

在X Server上Xorg是绝对地位，但是你使用的也可能是在Wayland规范上搞成的叫Xwayland的新X Server，存在的目的是保证过渡时的兼容性

在X Client上有XCB和Xlib，这里不介绍，它们之间还是有差异的

窗口管理器有KWin、 vtwm、Xfwm等，功能、风格各异，有的注重简洁高效，有的注重外观酷炫

### wsl

> wsl虽然可以运行linux桌面程序，但是有很多问题

1. 


### Windows

检测驱动是否正常

## 基本概念

一个窗口大概有这样的一些属性，它们在窗口被创建时指定

具体内容参见[文档](https://docs.rs/winit/latest/winit/window/struct.WindowAttributes.html)，跨平台性的文档具体参见[这个](https://docs.rs/winit/latest/winit/window/struct.WindowBuilder.html)

- 数值上的属性（也就是配置上是某个的属性）
    - 位置 （窗口左上角的位置）
    - 尺寸 （窗口的长款）   
    - 透明度 （默认是不透明的，但是通过修改这个属性这个可以做到透过这个窗口看见下面的窗口的效果    
- 布尔属性（也就是配置上是开/关的属性）
    - 能否缩放
    - 是否可见
    - 有无边框
    - 有无按钮
    - 是不是焦点（被选中的窗口）
- 其他属性
    - 标题 （显示在窗口边框上的）
    - 图标 （显示在任务栏上的应用的图标）
    - 颜色主题（也就是所谓暗色/亮色主题）
    - 全屏的模式（不是/独占屏幕/没有边框）
# 图形API

## 驱动问题

很多时候，驱动问题会导致图形运行地非常缓慢或者无法运行。这个问题在Windows和Linux都相当常见，在wsl更加常见(但是谁会用wsl整图形开发呢？ ~~~我~~~ )

首先，你得确定你的设备有显卡，如果你的设备没有显卡请移步软渲染

## Windows

可以打开任务管理器`Ctrl + Shift + Esc`，切换到`性能`选项卡，你应该可以找到`GPU0` `GPU1`之类的
![](./windows_gpu_device.png)

如果没有，大概率是有问题，你可以打开`设备管理器进一步确认`。打开`显示适配器`，你会看到这样的显示：

* 一个/多个 `Intel` `NVIDIA` `AMD` 开头的条目 总之不太可能是中文，如果是中文，大概率是有问题。如果图标右下角有黄色感叹后，也
![](./windows_device_manager.png)

关于Windows的驱动问题，已经有巨量的资料了，大概总结一下就是：

1. 如果是笔记本，看看厂家有没有提供驱动，如果有优先使用厂家提供的

2. 在官网下载驱动 [intel](https://www.intel.cn/content/www/cn/zh/support/detect.html) [amd](https://www.amd.com/zh-hans/support) [Nvidia](https://www.nvidia.cn/geforce/drivers/)，点击连接可以直接跳转

3. *最好不要使用什么驱动xx之类的软件，官网不能解决的问题它们就可以解决吗*

## Linux

Linux下的GPU驱动算是一个比较复杂的话题，我个人用的比较多的是[ArchLinux](https://archlinux.org)，所以以下经验基本是建立在ArchLinux上的

# GPU


## 图形API

你可能听说过`显卡(又叫Graphics Process Unit,GPU)`的概念，并且知晓显卡是用来渲染图形的，也就是负责绘制画面。

实际上，显卡就是一个构造比较特别的处理器，如果说CPU的计算能力是一个带学生团队，显卡的计算能力就是几百号小学生

显卡的设计是告诉进行大量计算，如果你看过多线程相关的知识，显卡就是有大量的物理线程，可以同时执行成千上万的运算

但是显卡在某些方面，比如内存延迟，分支预测等不优秀，而且不是每个运算单元都有自己的指令流水线，GPU内部的每一组运算单元只能运行一个程序

基本上，GPU是作为CPU的计算工具，辅佐CPU的计算工作罢了，GPU的运算任务也都是来自CPU的安排

就像CPU运行程序一样，GPU也是运行程序的，*所以你需要为GPU编写适用于GPU的程序*。用户是通过驱动来使用GPU，而现存GPU千千万，为了有通用的方法使唤GPU，就有了**图形API**

通过使用图形API，就可以通过驱动在不同的GPU上做一样的事，而不是为了每个GPU写一个代码

可以预料的，图形API是有很多个的，这里我会简单介绍一些

> 这里说到的*跨平台*起码包括Windows，Linux，MacOS，Android，IOS 这些平台

### OpenGL

> OpenGraphicsLibrary

元老级，是一个*跨平台*的图形API，用起来很*方便(存疑)*，并且简单易学。但是它受制于时代，设计有很大问题(有时用起来也有奇奇怪怪问题)

它并不能很好地压榨GPU性能，不够底层，相对来说可以做的事也比较少。很符合它的名字，他只是一个*图形库*，所以不可以用来进行*通用计算*

**它本尊于2017年停止更新**，但是它本身和它的分身(GLES,WebGL)仍在桌面，Web端和移动端等诸多领域活跃

~~它的分身也不会继续更新了！~~

不管怎么说，OpenGL还是图形开发的*首选*，对于Rust，它的绑定有底层的[glow](https://crates.io/crates/glow)和高级的[glium](https://crates.io/crates/winapi)。

### Vulkan

它是OpenGL的替代者

> 这并不是口号，它是OpenGL原班人马开发的，**OpenGL停止开发也是为了它**

Vulkan也是一*跨平台*的图形API，代表着*新一代图形API*(比如DX12,Matel)。被认为有着很高的学习成本，但是我觉得相比于OpenGL它需要额外学习的东西是一个开发者所必须具备的！

Vulkan不再是OpenGL那样的*状态机*设计，而是先进得多的*面向对象*。它暴露了底层的细节，拆分和定义了各种结构，把工作交给程序员，使得一切仅仅有条，使得它天生亲近多线程

虽然开发上多了一些麻烦，但是这说明你可以根据自己的想法控制一切，知晓背后的细节，而不会被奇奇怪怪的问题烦恼，这一点*很重要*

Vulkan的*计算管线*更使得*通用计算*成为可能，使得GPU可以从*图形处理单元*变成*纯粹的计算机器*

Vulkan原生支持C++和C，对于Rust有着[ash](https://crates.io/crates/ash)这样的优秀底层绑定，和[vulkano](https://crates.io/crates/vulkano)这样的上层绑定

### DirectX

> 很抱歉我对这个了解不多，我也没直接用过这个API

这是微软为Windows，Xbox等开发的一个图形API，它*只支持微软的平台*！

> 貌似它比一个图形API大很多？

我个人认为它对标的是OpenGL，一般它在Windows也有很好的性能。而*DX12*做出了很多改变，成为就像Vulkan一样的*新一代图形API*，一样暴露了底层的细节交给程序员

> 为什么有的游戏DX12性能拉跨？本质不是别的，就是程序员拉跨！

它的Rust绑定就是[winapi](https://crates.io/crates/winapi)的一部分

### WebGPU

***主角登场！***

> 在[WASM](https://developer.mozilla.org/zh-CN/docs/WebAssembly)之后，可以预料的是，对于Web平台也会有一个高性能地操作GPU的API来取代WebGL

WebGPU也是*新一代图形API*，只不过它是面向*Web平台*(浏览器)的，它使得Web开发者也可以高效地享受到主机的CPU资源。毕竟WebGL就是OpenGL分身之一，不能满足人们日益增长的性能需要

WebGPU使得Web上的通用计算和高性能渲染成为可能，它的性能几乎就是使用原生API，因为它运行的原理就是让各个浏览器开发商提供WebGPU调用到实际图形API的转译。WebGPU里的GPU资源什么的，也是实际的GPU的资源，没有半分虚假

那么如果，我是说如果，把API转移这一部分从浏览器分离出来，不就可以用WebGPU这个图形API开发支持多种图形API的程序吗？

WGPU就是干这个的，它使得开发者可以使用WebGPU API开发*跨平台*，*跨API*程序，这个*跨平台*还额外跨了Web平台！

一个WGPU程序，它可能用的是[*Vulkan*](#vulkan)，可能是[*DirectX*](#directx)，也可能是[*OpenGL*](#opengl)。这样你的程序就不会因为不支持某种特定的API而无法运行，你可以选择合适的*适配器*来使用不同API，或者自动选择，对于不同API你可以只写一次代码

有趣的是，对于Web目标，它同时支持WebGPU和WebGL API，使得在WebGPU普及之前就可以有基于WGPU的应用。它对于其他API的支持也是如此目的，它从来都不不是一个单纯的*WebGPU绑定*，不要被它的名字迷惑，更多意义上它已然是Rust最好的图形API！

接下来的教程使用的就是这个API

## 驱动

> 驱动只是开始而已

[ArchLinuxWiki](https://wiki.archlinux.org/title/Xorg#Driver_installation)说的很好，我就不扩充，总之就算是安装那些`xf86-video-`开头的软件包

但是， ~~有句古话说得好，**Nvidia, fuck you!**~~ 如果你是nvidia显卡，事情就变得有意思起来了！

nvidia显卡分为开源的闭源的，开源的是mesa实现的nouveau驱动
``` bash
sudo pacman -S xf86-video-nouveau
```
毕竟不是官方参与开发的，这个驱动只能说一般情况下ok，我也没碰见过问题。不过除此之外也有nvidia家的闭源驱动选择
> ***需要注意的是它会/需要禁用nouveau的加载，在archlinux这个过程自动完成***

> 以下内容可能只对archlinux适用

首先，你得知道你是什么内核

对于标准的`linux`内核，直接安装这个就可以，*不推荐通过官网提供的二进制程序安装这个*
```bash
sudo pacman -S nvidia
```
对于长期支持版的`linux-lts`内核，
```bash
sudo pacman -S nvidia-lts
```

但是，其他内核就比较麻烦了，只能用*DKMS(Dynamic Kernel Module System)(动态内核模块系统)*解决
> 每次变更内核，都会自动重新编译一次内核模块
```bash
sudo pacman -S nvidia-dkms
```

## mesa

mesa相当好，是一个开源的图形驱动，适用于大量GPU