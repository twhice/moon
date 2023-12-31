# Rust进阶


*先后顺序不完全体现在这里*

| 章节 | 介绍 |
| --- | --- |
| crates.io  | 成为第三方库开发者，在crates.io上传你开发的库或者程序，让所有人可以使用！ | 
| 多线程     | CPU那么多核心，只用一个太浪费了！学会多线程更好地压榨CPU，同时学会组织代码 |
| [异步](./async.md)       | 多线程不是万能的，有时还需要异步补足短板。为什么不放多线程里？因为这是一个很复杂的话题|
| rust高级特征   | 学习rust特征更高级的部分。这个特征指的是`trait`而不是汉语的特点的意思 |
| c          | 都学了rust，学下c语言应该轻轻松松吧？顺便补一下缺失的基础和c进行对比和学习，你将在unsafe rust的道路上取得长足的进步 |
| 特性揭秘   | 了解各种语法，特性的本质，这部分是相当可选，但是极大帮助的 |
| rust ffi   | ffi意思是`外部函数接口`，实现不同语言之间的调用，这部分极度底层 | 
| 声明宏     | 为了写出超级灵活的代码，进行更高的抽象，宏必不可少 | 
| 过程宏     | 如果可以用rust写rust呢？这样的宏超级灵活，但是代价是... |
| rust函数式 | rust的函数式特性，主要介绍闭包，迭代器等，从rustly中分出来是因为这很重要 |
| "rustly"   | 很多东西外部库都有实现，但是不用它们是你的选择。然而重复的实现rust标准库有的东西，写出不"rustly"的代码，对你我都不好。这部分会对std的实用部分，和代码规范进行介绍，写出更加rustly的部分 |
| 计算机图形学| 计算机三大艺术之一，感受线性代数的美丽！或者，学会使唤GPU干活，发挥它计算机器的用途 |

到这里，可能你会开始怀疑，会**多而不精**

我的回答是，会，但是只是短暂的会，因为这是电子书，这可以很方便地更新，而不是像出版书籍一个错误就永远和这本书相伴，相信时间的力量可以纠正和优化这本书