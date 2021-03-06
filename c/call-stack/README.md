# 函数调用与栈

从汇编中的子程序到 C 语言中的函数，再到面向对象语言中对象的方法，编程语言中复用代码段的能力在不断进步。但语法的演进背后的基础原理总是相似的，借助 C 语言理解函数调用的原理之后，能帮助你更轻松地以触类旁通的方式理解更高级语言中类似的机制。


## 函数的意义
为什么需要函数？在[控制流](../control-flow)一节中，我们已经能够编写各种复杂的逻辑判断流程，但还对于功能相近的代码，我们尚且没有一种稳定的方式来**复用**它们。而在 C 语言中，实现这个能力的概念就是所谓的函数了。

除了复用自己编写的代码之外，复用第三方库也是函数的重要用途之一。回想我们对 `printf` 函数的调用，它实际上就是一个库函数。通过将可复用的代码段封装为函数后作为库发布的方式，我们就不需要重复发明轮子，在大量库函数的基础上构建我们的上层应用了。


## 函数的背后
熟练的前端开发者会编写大量的函数。函数最基本的能力有哪些呢？不外乎下面这几点：

* 函数可以接受参数。
* 函数内可以定义局部变量。
* 函数可以返回值。
* 函数中可以调用其它函数。

从 C 到 JavaScript，函数的这几个基本能力是一致的。当然了，作为函数一等公民的 JavaScript，其闭包、匿名函数等能力是 C 原生不具备的。但要搞清楚 Objective-C 等 C -like 语言的方法、消息等概念，从 C 语言函数的原理出发是完全可行的。

另一方面，虽然 JavaScript 和 C 有着相近的函数语法和函数调用的机制，但巨大的区别在于 C 是一个非常贴近底层的语言，它的功能可以非常简单地对应到汇编代码中，而 JavaScript 则几乎屏蔽了全部的底层复杂度。因而在各类 JavaScript 的介绍性文章中，你多半只能对背后的执行机制有个道听途说的认识，无法获得直观而切实的感受。但在 C 里要讲清楚这些内容就容易得多了。让我们从 C 的函数语法开始吧：

``` c
// 声明
int add (int a, int b);

// 定义
int add (int a, int b) { return a + b; }
```

C 的函数并非一等公民，函数需要先声明，后使用。函数声明既可以放到头文件中以便于代码的复用，也可以简单地写在 `.c` 代码的顶部。

我们已经知道，if 和 for 等语句编译成汇编后，只相当于在其中的代码块首尾加上几条判断和跳转指令而已。函数编译成汇编时也可以这么简单地实现吗？我们需要注意一个重要的机制：作用域。

C 中的函数和 if for 等控制流语句的一大区别，在于函数内部有自己的作用域。譬如这样的代码是完全合法的：

``` c
int x;

if (...) {
  x = 10;
}
```

但对于函数而言，下面的代码就违背了作用域机制：

``` c
int x;

void fn () {
  x = 10;
} 
```

当然了，JavaScript 中通过闭包可以很方便地访问到函数外层的变量。但为什么在 C 中函数的大括号里就找不到外层的变量呢？我们如果查看 [call.c](./call.c) 编译成的汇编码，会发现函数调用时所使用的指令不是 `JMP` 而是 `CALL`，这有什么不同呢？

和 `JMP` 纯粹地跳转到某一个代码段继续执行不同，`CALL` 不仅仅能跳转到另一个代码段位置，更能够在接下来遇到 `RET` 指令时，直接回到 `CALL` 所在的位置继续执行。联想一下 C 中函数代码段尾部的 `return`，不难发现一个 C 的函数体对应到汇编，也就是被 `CALL` 和 `RET` 指令包裹起来的一块代码而已。

这和作用域有什么关系呢？之前的控制流结构里，`JMP` 这样的指令基本上都是在相邻的几段代码之间来回跳跃，不会跳转到一个很遥远的位置。这也就带来一个问题：在 `CALL` 过去的位置，怎么样传递函数的参数呢？汇编的寄存器只有那么几个，但 C 中的函数至少支持 127 个参数，这是怎么做到的呢？并且，`CALL` 过去的代码段里完全可以继续 `CALL` 其它地方，然后再用多个 `RET` 逐次返回。这里面也没有对函数参数长度的限制。并且，后 `CALL` 的代码段会先被 `RET` 回去，有什么数据结构能够满足这种需求呢？

等等，可变长度和后进先出，这不就是经典的**栈**吗？基于栈的模型，我们需要一段连续的地址空间，每次函数调用的多个参数依次 push 到栈上，在函数体内部也通过栈上的偏移量来访问这些参数，并在函数返回时 pop 出栈，将内存空间释放。这样一来，多个嵌套的函数调用，就转化为了在栈上对地址线性地来来回回的操作。这样一个逐层存储函数调用参数的栈，就是我们耳熟能详的**调用栈**了。譬如 A 函数中调用了 B，而 B 中调用了 C，那么调用栈应该看起来像这样：

``` nasm
---
C
---
B
---
A
---
```

关于调用栈有一个反直觉的地方：A 虽然在调用栈的底部，但它在内存中的地址一般却是最大的。这背后有些历史原因，但更接近车道是靠左行还是靠右行一样只是一个约定，故而虽然有所谓的“栈内存从高地址向低地址增长”这一说，但这个定义并不是特别准确。

既然我们已经引出了调用栈的概念，那么调用栈上放着的内容又是什么呢？调用栈上的每一项，都装着为一个函数所传入的实际参数，每个函数所对应的项，我们称之为**栈帧**。每个栈帧里放置的实际参数，其实也就是函数里能够访问得到的局部变量了。

现在我们可以把函数的能力和底层的原理对应起来了：

* 函数可以接受参数 - 参数逐个存在栈帧里。
* 函数内可以定义局部变量 - 函数内只能访问栈帧上的数据。
* 函数可以返回值 - 对应汇编的 `RET` 指令。
* 函数中可以调用其它函数 - 调用函数，相当于 push 一个新帧；函数返回，相当于 pop 出栈顶的帧。

走通这些概念后，不妨思考这个问题：函数调用栈有专门的位置存放吗？有的，这段空间用 `RSP` 和 `RBP` 寄存器表示（**R**eserved **S**tack **P**ointer 与 **R**eserved **B**ase **P**ointer）。`RSP` 指向当前栈顶，`RBP` 指向当前栈帧底部。函数调用时的简化汇编代码大致形如：

``` nasm
PUSHQ   %RBP        ; 保存上一个栈帧地址
MOVQ    %RSP, %RBP  ; 设置 RBP 为当前栈顶
SUBQ    $16, %RSP   ; 为局部变量留出 16 字节

; ...函数体

MOVQ    %RBP, %RSP  ; 重置 RSP 到栈底
POPQ    %RBP        ; 恢复上一个栈帧
```

到这里，相信你已经可以理解调用栈的概念了。看起来虽然有些费劲，但在接下来的内容中它对我们很重要 :)
