# Virtualization

## 4.线程的抽象

- 线程的定义：正在运行的程序。
- 问题来了：如何提供有许多CPU的假象？
- 答：通过虚拟化的技术来制造有许多CPU的假象。即通过**time sharing**的方式。
- 为了实现虚拟化，OS提供了低层和高层的一些机制。
- 低层上主要是提供用来实现功能的方法和协议。比如，提供了如何实现**context switch**来使OS停止运行一个程序从而运行另外一个程序。
- 高层上主要是提供了一些智能的policies。policies其实就是一些OS用于做决定的算法。比如说现在有一堆程序，先让哪个程序上CPU运行？在OS上有一系列的算法来决定这些事情。

### 4.1 什么是线程？

- 操作系统提供一种抽象被称为线程，用于运行程序。
- 为了明白线程是有什么组成的，我们必须先谈谈**machine state**：即当一个程序处于运行时，它可以读取或者更新什么？
- 第一个组成线程的**machine state**就是内存(**memory**)。指令存在内存中，程序读写的数据存在内存中。
- 第二个组成线程的**machine state**是寄存器(**registers**)。许多指令都读取或者更新寄存器，因此寄存器很重要。
  - 有许多特殊的寄存器，比如**program counter(PC)**，用于表明程序的下一条指令在哪里；**stack pointer**和**frame pointer**用于管理函数的参数、变量和返回值的地址。
- 第三个组成线程的**machine state**是**I/O information**。其中包含了一系列的进程正在打开的文件。

### 4.2 进程的接口

- 以下接口为OS必须为进程提供的一些接口：
- **Create**：OS必须提供一些方法来创建新的进程。
- **Destory**：既然我们可以创建，那必须可以强制销毁一个进程。正常情况下，进程应该主动退出，但是一旦有异常发生，我们需要强制销毁一个进程。
- **Wait**：当一个进程停止运行的时候，OS应该提供wait接口。
- **Miscellaneous Control**：相当于是一些操作的结合体，比如先停止一个进程，等待一会后再将其恢复运行。
- **Status**：用于查看一个进程的信息，比如运行了多久或者出于什么状态。

### 4.3 进程创建的细节

- 提问：如何将一个程序转变为进程？更明确的问，OS如何启动或停止一个进程？进程创建是如何工作的？
- 第一，在OS运行一个程序前需要从磁盘（或者固态）中，加载源代码和静态数据到内存中（或者叫进程的地址空间）。
  - 早期OS中，加载线程是热加载，在运行前全部加载进来；现代OS一般是懒加载，只加载程序运行需要的部分。

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220322212127.png" style="zoom: 50%;" />

- 第二，在加载完源代码和静态数据后，OS需要将一部分内存分配给进程的**stack**。比如在C中，局部变量，函数参数，返回值的地址都是在stack中。
- 第三，除了要给程序分配**stack**，还要给程序分配**heap**，**heap**用于动态分配的数据。比如通过malloc()函数分配、free()函数销毁的空间，列表、哈希表、树之类的数据结构。因为是动态分配的，所以heap一开始可能很小。
- 第四，OS还需要做一些和I/O有关的初始化。比如在Unix中，每个进程默认都有三个打开文件描述符，用于标准输入、输出、错误。
- 在经历过加载源代码和静态数据、分配stack、分配heap以及I/O初始化之后，finally OS把执行程序的环境搞好了。但是还有最后一件事，让程序在entry point处开始运行，也就是main()。在跳到main()入口后，OS终于让这个程序上了CPU，开始其进程的一生。



### 4.4 进程的状态

- 一般来说有以下三种最基础的状态：
- Running：进程正在处理器上运行。
- Ready：进程已经准备好上CPU，但是OS还允许它上。
- Blocked：由于进程需要完成某些操作，导致它还没有Ready。比如在等待I/O操作。

- 状态之间的转换：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220322212315.png" style="zoom:67%;" />

### 4.5 数据结构

- OS也是程序，因此它有一些数据结构。比如，为了追踪进程的状态，OS会保存running、ready、blocked的process list。
- 比如在xv6中，OS需要追踪的数据结构：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220322213001.png" style="zoom:67%;" />

- 其中保存了 **register context**，用于保存一个停止进程的寄存器内容。当一个进程停止时，其寄存器信息会保存在内存中，只要将寄存器恢复，那么进程就能回复继续运行，也就是后面会说的**context switch**。

- Process list用于保存正在运行的进程的信息，是一种比较简单的数据结构。还有别的比如说**PCB**(Process Control Block)。

### 4.6 总结

- 我们知道了OS最基本的抽象：进程，它是运行着的程序。
- 知道了进程，再来看看本质：低层的mechanisms和高层的policies，mechanisms用于怎么实现，policies用于如何规划。二者结合在一起我们才能理解OS如何虚拟化CPU。