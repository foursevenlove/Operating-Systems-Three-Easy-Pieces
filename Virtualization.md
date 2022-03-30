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



## 6.Mechanism: Limited Direct Execution

- 受限制的直接执行。
- 虚拟化CPU的基本思想：让一个线程上CPU运行一会，下来再让另一个上CPU运行，通过**time sharing**的方法来实现。
- 两个问题：
  - 第一是性能：实现虚拟化时如何能让系统不用额外的开销？
  - 第二是控制：如何在高效运行线程时OS可以重新获得对CPU的控制？

### 6.1 Limited Direct Execution

- 为了解决上述两个问题可采用Limited Direct Execution的方法。
- 先看什么是Direct Execution：单纯地让程序直接运行在CPU上。按照第四章的说法，OS需要在process list中创建进程入口，分配内存，加载代码到内存中，定位程序入口（main函数之类的），然后开始运行，流程如如下：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220325091511.png" style="zoom:67%;" />

- 这种方法的问题来了：
  - 第一，如果像这样仅仅运行程序，OS怎么能够确保程序不做一些OS不想让程序做的事？
  - 第二，在运行程序时，OS如何停止当前进程，让另外一个进程上CPU运行，从而实现**time sharing**来虚拟化CPU呢？
- 所以我们应该再谈谈"limited"。

### 6.2  Problem #1: Restricted Operations

- Direct Execution是很快没错，但是有个问题：如果进程想执行一些受限制的操作该怎么办？例如提出I/O请求来读写磁盘或者进程想使用更多的系统资源如CPU或内存？也就是说如何能够让进程去执行这些操作，但是又不把整个系统的控制权交给该进程？

- 先看看一种可行但是不安全实际也不会使用的方法：当进程运行时，让该进程可以执行它想执行的相关操作。明白人都知道，这种做法很不安全。因为这样该进程就可以读写整个磁盘，还谈何保护呢？
- 所以我们来看一种实际可行的方法。将处理器也就是CPU分成两种模式，**user mode**和**kernel mode**。
  - **user mode**：运行在user mode的代码的操作是受限的。比如在user mode的时候，进程是不能发起I/O请求的，如果它这么做了，就会导致异常，OS就会杀死该进程。
  - **kernel mode**：OS运行在kernel mode下，在这个模式下的代码可以执行他们想执行的任何操作，包括一些特权操作，比如发起I/O请求操作等。
- 问题又来了，在这种设定模式下，用户的线程想执行特权操作的时候，比如读写磁盘，那应该怎么办呢？
  - 解决方法：实际上现代硬件，都给用户的程序提供了执行**system call**的能力。
  - **system call**实际上给user mode的进程提供了一种方法，用于访问只有在Kernel mode才能访问的资源。
- 执行system call，程序必须执行一条叫**trap**的指令。这条指令用于转换到Kernel mode，这样系统就可以执行该程序想做的特权操作了。完成之后，OS调用一条叫**return-from-trap**的指令，返回到该程序，并且返回user mode。
- 执行trap的时候，硬件要做一些事情来确保保存好进程的寄存器，这样当OS调用return-from-trap的时候，才能恢复该进程的信息。例如在X86中，处理器会将程序的寄存器、flags、PC保存到每个程序的**kernel stack**中，当return-from-trap指令返回时，再从栈中pop出保存的值用于回复进程信息。其他系统做法不同，但核心思想相同。

---

- 现在我们解决了和权限有关的问题，但是新的风暴已经出现，那就是trap指令如何知道，在换到kernel mode的时候OS要运行什么代码呢？明白人又知道了，那肯定不能由发起trap指令的进程说了算，因为这不安全。
- 解决办法就是**trap table**。**trap table**在启动时就设置好了。机器在启动时，在kernel mode下设置，因此可以根据硬件需要来配置机器。**trap  table**里设置了当一些trap指令被调用时，应该运行什么代码来处理。比如当键盘输入或者有磁盘终端时应该做些什么。那么OS就让硬件记住了这些**trap handlers**的位置，硬件会记住这些**trap handlers**的位置直到下次重启。这就回答了上一个问题，当系统调用和其他的异常时间发生时硬件应该去执行什么代码。
- 系统调用千千万，如何直到当前进程想发起哪个系统调用呢？答案是**system-call number**，用于指定系统调用。
- **trap table** 有了，硬件也可以去记住他，但是怎么告诉硬件trap table在哪里呢？不能说让用户来告诉硬件吧？所以明白人又懂了，这也是一个特权操作。

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220326161917.png" style="zoom:80%;" />

- 图6.2表明了整个工作流程。在Limited direct execution（**LDE**）中有两个阶段。
  - 第一阶段，kernel初始化trap table，CPU记住它的位置，以便后续使用 。
  - 第二阶段，kernel做些设置以便准备让程序上CPU运行，然后执行**return-from-trap**，转换到user mode，程序开始运行。如果程序想执行特权操作，那么就要发起system call，然后执行trap指令，保存当前进程状态，然后把CPU还给OS，进入kernel mode，执行特权操作。然后就是返回，恢复进程，进程正常结束，OS释放内存。

### 6.3 Problem #2: Switching Between Processes

- 我们已经直到了如何保护整个系统，程序在CPU上运行的流程应该时怎么样的，现在就该考虑下另一个问题了。我们想要虚拟化CPU，那就要在不同进程之间切换上CPU，那么如何做呢？
- 想想好像很简单，先让一个下去再让另一个上来呗。当一个进程在CPU上运行时，OS是没有在CPU上运行的。但是要让一个进程上CPU又必须是由OS指定的，但是OS不在运行，这怎么办呢？
- 所以要解决的问题就是，要想让别的进程上CPU就要让OS重获对CPU的控制。

#### 6.3.1 A Cooperative Approach: Wait For System Calls

- 要想解决上述问题，一种过去的OS采用的做法是cooperative的方法。这种方法下，OS是信任进程的，OS相信进程可以规范他们的行为。也就是说运行太长的进程会定期地放弃CPU，通过系统调用；或者当程序出错的时候，会执行trap指令，这样CPU就重获了对CPU的控制。但是如果程序死循环呢？如果程序永远也不愿意下来呢？

#### 6.3.2 A Non-Cooperative Approach: The OS Takes Control

- 靠进程自觉不行，那就强制要求呗。如果不强制要求，进程陷入死循环永远不下CPU你就只能重启机器了。
- 这里我们就采用**timer interrupt**的方法。也就是说一个进程没运行一段时间就必须中断，这样它就下CPU了。当该中断发起时，就代表你该下了，另外一个进程该上了。
- 也就是说像之前讨论的那样，机器启动时，OS就告诉了硬件当timer interrupt的时候应该执行什么代码。在启动时，OS就起一个timer，这样OS才能重获对CPU的控制。
- 当timer interrupt的时候，应该保存当前进程的信息，比如寄存器信息等，这样下次该进程才能被恢复重新上CPU。

#### 6.3.3 Saving and Restoring Context

- 当timer interrupt时，通过scheduler（会在之后讲）来决定是当前进程继续上还是换一个上。如果是换一个进程上的话，那么也就是执行**context switch**操作，这是由低层的代码实现的。
- context swich的本质就是保存信息并且恢复信息，即保存当前进程的一些信息，恢复即将上CPU的另一个进程的信息。首先将当前进程的寄存器信息保存到进程自己的kernel stack中，然后再恢复下一个进程的kernel stack信息。
- 当然OS还要保存下别的信息，保存当前进程的通用寄存器、PC以及该进程的kernel stack指针，然后恢复下一个进程的这些信息。
- 流程如下：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220326165400.png" style="zoom:80%;" />

### 6.4 有关并发

- 聪明的你可能会问了，如果在处理中断的时候另一个中断发生了怎么办？其实这是之后在并发章节会讨论的事情，现在只需要直到当处理中断的时候，是不会允许其他中断的。

### 6.5 总结

- 这章的重点就是，第一程序如何运行，第二进程怎么切换。
- 我们通过LED来实现虚拟换CPU，其实就是让一个程序上CPU，但是留一手，通过timer让它不能一直运行。
- 那么如何选择下一个上CPU的进程？这是之后的章节讨论的问题了。



## 7.Scheduling: Introduction

- 前面我们了解了低层的用于运行进程的机制，现在该来谈一谈高层的policy来决定如何调度进程上CPU了。这章就来学习下调度进程的算法。

### 7.1 Workload Assumptions

- 在往下走之前，我们先做一些假设，有关于在系统中运行的进程的假设，这些假设我们叫做**workload**。当然我们做的这些假设其实在OS中是不切实际不现实的，但是随着深入下去，我们会逐渐接近实际情况，逐渐打破这些假设。
- 我们也经常把进程叫做**job**，对job我们做出如下五个假设：
  - 1.每个job运行时间相等
  - 2.所有job都在同一时间到达
  - 3.一旦job开始运行，就运行到job完成
  - 4.所有的job都只使用CPU（也就是说没有I/O请求）
  - 5.Job的运行时长已知

### 7.2 Scheduling Metrics

- 既然是调度算法，那么就要有一个调度指标，根据这个调度指标来决定让哪个进程上CPU。
- 首先我们使用一个最简单的调度指标：**turnaround time**（周转时间）。周转时间的定义为，job的完成时间减去job到达系统的时间：$$T_{turnaround}=T_{completion} - T_{arrival}$$。因为我们假设所有job同时到达系统，也就是说$$T_{arrival}=0$$，因此 $$T_{turnaround}=T_{completion}$$。
- 注意，周转时间是一种**performance**的指标，是我们这一章首要考虑的。另一种指标考虑的是**fairness**。这两种指标好比鱼与熊掌，不可得兼。

### 7.3 First In, First Out (FIFO)

- 最基本的算法**First In, First Out (FIFO)**，有时也叫 **First Come, First Served (FCFS)**。过于简单不多赘述。优点是简单易实现。
- 假设A、B、C几乎同时到达，但是FIFO是挑一个先到达的，所以假设A比B先一点，B比C先一点。

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220328161849.png" style="zoom:80%;" />

- 此种情况下，计算每个job的平均周转时间：$$\frac{10+20+30}{3}=20$$。
- 现在让我们打破第一个假设，即不再认为所有job都运行相同的时间，这种情况下FIFO的性能有时就不是很好。例如以下情况：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220328162231.png" style="zoom:80%;" />

- 此种情况下，计算每个job的平均周转时间：$$\frac{100+110+120}{3}=110$$。
- 这种问题叫 **convoy effect**（护航效应），就是说对资源占用较少的job排在了对资源占用较多的job后面。那咋办呢？且看下一种算法。

### 7.4 Shortest Job First (SJF)

- 短作业优先，顾名思义，运行时间短的job先上CPU运行。
- 在之前的例子上，可得到如下图运行顺序：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220328211902.png" style="zoom:80%;" />

- 计算每个job的平均周转时间：$$\frac{10+20+120}{3}=50$$。

- 在我们假设所有job同时到达的情况下，SJF是一个优化的调度策略。但实际情况下往往不是所有job同时到达的。
- 现在让我们打破假设2，即job有可能在任何时刻到达，这种情况下应该怎么办？如果用SJF可能会出现以下情况，A先到，B、C在10s到，且A运行100s，B、C分别运行10s：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220328213101.png" style="zoom:80%;" />

- 这时每个job的平均周转时间：$$\frac{100+(110-10)+(120-10)}{3}=103.33$$。

### 7.5 Shortest Time-to-Completion First (STCF)

- 为了解决上面这种情况，考虑到我们之前学过的timer interrupt和context switch我们采用新的算法**Shortest Time-to-Completion First (STCF)**或者叫**Preemptive Shortest Job First (PSJF)** 。也就是抢占式的SJF，即当一个job到达时，如果它的运行时间短，那可以抢占当前job正在使用的CPU。
- 在之前的例子上，如下图：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220328213624.png" style="zoom:80%;" />

- 这时每个job的平均周转时间：$$\frac{(120-0)+(20-10)+(30-10)}{3}=50$$。

### 7.6 A New Metric: Response Time

- 如果我们知道job的运行时间并且job仅仅使用CPU的话，那么周转时间是我们唯一的指标，STCF是一个很好的策略。但事实上，坐在终端前的用户希望能够和系统进行交互，因此我们必须考虑另一个指标：**response time**（响应时间）。
- 响应时间的定义为job第一次上CPU的时间减去job到达系统的时间：$$T_{response}=T_{firstrun} - T_{arrival}$$。
- 对图7.5，A的响应时间为0，B为0，C为10。
- 之前的STCF对于考虑周转时间的情况下是很好的，但是却不利用响应时间，你想想你要是C的用户，你的程序要10s才能得到响应，你急不急？所以我们应该使用一种对于响应时间敏感的算法。

### 7.7  Round Robin

- 为了解决上述问题，采用 **Round Robin（RR）**的算法。基本思想就是不让每个job运行到它技术，每个job在CPU上都只能运行一个时间片（**time slice**有时也叫**scheduling quantum**），然后换下一个job上来运行。重复以上直到所有job运行结束。注意，时间片的长度必须是timer interrupt的倍数。
- 举个栗子，A、B、C同时到达，每个都要运行5s。SJF算法运行图如7.6，RR算法运行图如7.7：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220328215527.png" style="zoom:80%;" />



- RR算法每个job的平均响应时间：$$\frac{0+1+2}{3}=1$$，SJF算法每个job的平均响应时间：$$\frac{0+5+10}{3}=5$$，只能说高下立判。
- RR算法中，时间片的长度太长或者太短都不行。
  - 如果太短了，那么context switch的代价会影响整个性能。
  - 如果太长了，那么回到了老问题，响应时间太长。
- 举个栗子，还是上面的ABC，看图7.7，A在13s结束，B在14s结束，C在15s结束，其实很差劲。如果考虑周转时间的话，RR真的拉跨。又回到了那句话，performance和fairness不能得兼。

### 7.8 Incorporating I/O

- 现在该打破第四个假设了，也就是说程序是需要I/O操作的。
- 那么程序在等待I/O的时候，其实CPU是空闲的，进程是blocked的。那么这时就应该安排别的进程上CPU，等待I/O的进程等到I/O结果了，再恢复该进程。
- 举个栗子，A、B各需要50msCPU，区别在于，A每10ms就发起一次I/O请求，B不需要I/O。

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220329000554.png" style="zoom:80%;" />

- 假设我们用的是STCF，很明显，这时CPU的利用率是很低的。所以应该充分利用CPU，像下面这样：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220329000716.png" style="zoom:80%;" />

### 7.9 No More Oracle

- 现在该打破最后一个假设了，也就是说调度器不知道每个job需要运行时长。这个时候SJF/STCF都不好用了，RR也是一样。别着急后续会讲。

### 7.10 Summary

- 两个调度指标周转时间和响应时间，不可得兼。
- 针对两者，对应有不同的算法。





## 8.Scheduling: The Multi-Level Feedback Queue

- 书接上回，我们来看看如何平衡两个调度指标，即周转时间和响应时间。
- 采用 **Multi-level Feedback Queue (MLFQ)**，解决了两个问题：
  - 第一，优化了周转时间。
  - 第二，同时兼顾了响应时间。

### 8.1 MLFQ: Basic Rules

- MLFQ有不同的实现方式，但是基本思想是一致的。
- 在我们的方法中，MLFQ有多个队列，每个队列有不同的优先级。在任意之间点，处于ready状态的job只能处于一个队列中。MLFQ根据队列的优先级来决定运行哪一个job：优先级高的job可以先上CPU运行。当然，一个队列中可能有多个job，那么我们就采用RR的算法来运行处于统一队列中的job。
- 因此，MLFQ的两条基本规则：
  - **Rule 1**: If Priority(A) > Priority(B), A runs (B doesn’t) 
  - **Rule 2**: If Priority(A) = Priority(B), A & B run in RR
- 举个栗子：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330181943.png" style="zoom:80%;" />

- 在上图中，由于A、B优先级高，所有A、B会以RR的方式轮流使用CPU，而可怜的C和D永远也使用不了CPU--an outrage！
- 所以咋办？答案就是优先级不能是定死的，也就是说每个job的优先级是需要动态改变的。那根据什么规则来改变呢？可以根据每个job的历史行为来预测它未来的状态，根据历史来改变它的优先级。比如，如果一个job一直在让出CPU，因为它在不停等待键盘输入，那么就要提高它的优先级，因为当前该job正在和用户交互，要缩小响应时间；再比如，如果一个job长时间地使用CPU，那么就要适当减低它的优先级，因为要考虑到别的job。

### 8.2 Attempt #1: How To Change Priority

- 在给出答案之前，我们明确：需要交互式的job一般是运行时间短的（并且有可能会经常退出CPU），运行时间长的job一般需要长时间的CPU也就是说响应时间没那么重要。
- 这里尝试第一次给出改变优先级的规则：
  - **Rule 3**：当一个job到达系统的时候，被放进优先级最高的队列。
  - **Rule 4**：如果一个job在运行时，使用完了分给它的整个时间片，那就降低它的优先级（把他移到下一级队列）。
  - **Rule 5**：如果一个job在时间片用完之前就让出了CPU，那么保持它的优先级不动。
- 举个栗子：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330183313.png" style="zoom:80%;" />

- 如果有短job来了咋办？再举个栗子：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330183359.png" style="zoom:80%;" />

- I/O咋办？再举个栗子，灰色job不停的需要I/O操作。

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330183706.png" style="zoom:80%;" />

- 现在的MLFQ的问题：
  - 第一，**starvation**，也就是饥饿。如果系统中存在大量的交互性job，也就是说有很多的job，它们由于需要I/O操作一直停留在优先级高的队列中，那么优先级低的队列中的job永远得不到执行，饿死啦！
  - 第二，**game the scheduler**，也就是恶意程序把调度器骗了。想象一下，如果一个job在使用完99%的时间片后就发起一次I/O请求，根据我们的**Rule5**，因为它没有使用完时间片所以它会保持优先级不变，那么它就有可能永远停留在优先级最高的队列中。
  - 第三，一个job的行为是可能发生**变化**的。比如说，一个job可能前期大量使用CPU，后期就变成需要交互性的了。因为之前它大量使用CPU，可能它已经到了优先级最低的队列，但此时它又需要大量I/O和用户交互，那么此时的响应时间就会很高。

### 8.3 Attempt #2: The Priority Boost

- 为了解决上述问题，我们重写**Rule 5**，思想很简单，每隔一段时间增加所有job的优先级。做法有很多，我们采取最简单的：每隔一段时间，就把所有的job都放到优先级最高的队列中。

  - **Rule 5**：在周期**S**后，将系统中的所有job移到优先级最高的队列中。

- 这种做法解决了上述的两个问题：

  - 第一，解决了饥饿问题。看下图，左边没有增加优先级的时候，黑色job永远待在Q0，暗无天日。右图有了增加优先级，每隔一段时间所有job都会被移到优先级最高的队列，因此黑色job有了出头之日。

    <img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330190852.png" style="zoom:80%;" />

  - 第二，解决了job的行为会发生变化的问题。如果一个job早期大量使用CPU，在后期又转变成了交互型的，那么由于这个规则，每隔一段时间它就被移到优先级最高的队列，因此该问题解决。

- 那么**S**该如何设置？如果S过长，那么还是可能有饥饿现象，太短的话，交互型job可能不能很好的共享CPU。（一般采用黑魔法， 不懂）

### 8.4 Attempt #3: Better Accounting

- 上面的做法很好，解决了两个问题，但是还有一个问题没有解决。如何应对恶意程序？
- 我们重写Rule 4a和Rule 4b，变成：
  - **Rule 4**：只要一个job使用了所在队列分配的一个时间片的时间，那么就降低它的优先级（不管它有没有在使用完一个时间片之前让出CPU）
- 举个栗子：没有这种机制时，灰色程序就一直可以停留在最高级别的队列，

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330192159.png" style="zoom:67%;" />

- 有了这种机制，那么就变成：

<img src="https://raw.githubusercontent.com/foursevenlove/gitResource/master/Typora20220330192227.png" style="zoom:67%;" />

### 8.5 Tuning MLFQ And Other Issues

- MLFQ还有一些问题，即如何设定调度器的参数？比如有多少个队列？每个队列的时间片长度应该是多少？定期增加优先级的周期应该是多少？等等这些问题，很不好解决，只能根据一些经验来设定，所以不赘述。

### 8.6 MLFQ: Summary

- 优化后的MLFQ的规则：
  - **Rule 1**：如果A的优先级大于B，A可以运行。
  - **Rule 2**：如果A、B优先级相等，A、B以RR的方式运行，时间片的长度由所在队列的等级决定。
  - **Rule 3**：一个job进入系统时，放到最高级别的队列中。
  - **Rule 4**：只要一个job使用了所在队列分配的一个时间片的时间，那么就降低它的优先级（不管它有没有在使用完一个时间片之前让出CPU）
  - **Rule 5**：在周期**S**后，将系统中的所有job移到优先级最高的队列中。
- 可以看出，MLFQ不需要知道一个job需要运行多久，而是根据job的历史行为来预测未来的行为；并且兼顾了周转时间和响应时间。