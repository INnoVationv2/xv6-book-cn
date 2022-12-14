# 第二章 操作系统的组织结构

操作系统的一个关键要求是要同时支持几个活动。例如，使用第一章中描述的系统调用接口，一个进程可以用fork启动新的进程。操作系统必须在这些进程之间采用时分复用分享计算机的资源。例如，即使进程的数量超过了硬件CPU的数量，操作系统也必须确保所有的进程都有机会执行。操作系统还必须保证各进程之间的隔离。也就是说，如果一个进程发生故障，它不应该影响其他不依赖故障进程的进程。然而，完全的隔离又太过了，因为进程可能有意地进行交互；管道就是一个例子。因此，一个操作系统必须满足三个要求：复用、隔离和交互。

本章概述了操作系统是如何组织以实现这三个要求的。事实证明，有很多方法可以做到这一点，但本文重点介绍以[宏内核](https://zhuanlan.zhihu.com/p/54283709)为中心的主流设计，许多Unix操作系统都采用这种设计。本章还概述了xv6进程，它是xv6中的基本隔离单元，以及xv6启动时第一个进程的创建。

Xv6运行在一个多核1的RISC-V微处理器上，它的许多底层功能（例如，它的进程实现）是RISC-V特有的。RISC-V是一个64位的CPU，xv6是用 "LP64 "C语言编写的，这意味着C编程语言中的long（L）和指针（P）是64位的，但int是32位的。本书假定读者已经在一些架构上做了一些机器级的编程，并将在出现RISC-V特定的想法时介绍这些想法。RISC-V的一个有用的参考资料是《The RISC-V Reader: An Open Architecture Atlas》。《The user-level ISA》和《the privileged architecture》是官方规范。

> "多核 "是指多个共享内存但并行执行的CPU，每个CPU都有自己的寄存器。本文有时使用多处理器一词作为多核的同义词，尽管多处理器也可以更具体地指具有几个不同处理器芯片的计算机。

一台完整的计算机中，CPU被辅助它的硬件所包围，其中大部分是I/O接口。Xv6是使用qemu的"-machine virt "选项所模拟出的硬件编写的。这包括RAM、包含启动代码的ROM、与用户键盘/屏幕的串行连接，以及一个用于存储的磁盘。

### 2.1 物理资源的抽象

当遇到一个操作系统时，人们可能会问的第一个问题是为什么要有这个系统？也就是说，我们可以把图1.2中的系统调用实现为一个库，应用程序与之链接。在这个方案中，每个应用程序甚至可以有自己的库来满足自身需求。应用程序可以直接与硬件资源交互，并以最适合应用程序的方式使用这些资源（例如，实现高的或可预测的性能表现）。一些用于嵌入式设备或实时系统的操作系统就是以这种方式组织的。

这种方法的缺点是，如果有多个应用程序在运行，这些应用程序必须运行良好。例如，每个应用程序必须定期让出CPU，以便其他应用程序能够运行。如果所有的应用程序都相互信任并且没有错误，这样的合作式时间共享方案也许是可行的。但是更常见的情况是，应用程序之间互不信任，并且有漏洞，因此人们通常希望得到比合作式方案更强的隔离。

为了实现强大的隔离，禁止应用程序直接访问敏感的硬件资源，将资源抽象为服务是很有效的。例如，Unix应用程序只能通过文件系统提供的`open`、 `read` 、`write`和`close`api与存储系统交互，而不是直接读写磁盘。应用程序可便利的直接使用路径名进行访问，并允许操作系统（作为接口的实现者）来管理磁盘。即使不考虑不隔离带来的问题，需要与磁盘进行交互的程序（或者只是希望不妨碍其他程序对磁盘的访问）也会发现使用抽象的文件系统比直接使用磁盘更加方便。

同样，Unix在进程之间透明地切换CPU，必要时保存和恢复寄存器状态，这样应用程序就不必自己考虑分时共享。这种透明度使操作系统能够共享CPU，即使一些应用程序处于无限循环之中。

作为另一个例子，Unix进程使用`exec`来建立它们的内存镜像，而不是直接与物理内存进行交互。这允许操作系统自行决定在内存中放置进程的位置；如果内存紧张，操作系统甚至可能将进程的一些数据存储在磁盘上。`exec`还为用户提供了便利的文件系统，以存储可执行程序的镜像。

Unix进程之间许多形式的交互是通过文件描述符进行的。文件描述符不仅抽象了许多细节（例如，管道或文件中的数据存储位置），而且还以一种简化交互的方式来定义它们。例如，如果管道中的一个应用程序失败了，内核就会为管道中的另一个进程发送一个EOF信号。

图1.2中的系统调用接口是经过精心设计的，既为程序员提供了方便，也提供了强隔离的可能性。Unix接口并不是抽象资源的唯一方法，但是它已经被证明是一个非常好的方法。

### 2.2 用户模式(目态)、监督者模式(特权模式、管态)和系统调用

> 目态和管态翻译的够抽象的，管态还可以将就着理解为管理态，目态实在无法理解。

强隔离需要在应用程序和操作系统之间有一个硬边界。如果应用程序运行出错，我们希望操作系统或其他应用程序不能受到影响。相反，操作系统应该能够清理失败的应用程序并继续运行其他应用程序。为了实现强隔离，操作系统必须做到应用程序不能修改（甚至不能读取）操作系统的数据结构和指令，并且应用程序不能访问其他进程的内存。

CPU为强隔离提供了硬件支持。例如，RISC-V有三种模式，CPU可以在其中执行指令：机器模式、监督者模式和用户模式。在机器模式下执行的指令具有完整的权限；CPU以机器模式启动。机器模式主要是用来配置计算机的。Xv6在机器模式下执行了几条指令，然后转到监督者模式。

在监督者模式下，CPU被允许执行特权指令：例如，启用和禁用中断，读写保存页表地址的寄存器，等等。如果用户模式下的应用程序试图执行一条特权指令，那么CPU不会执行该指令，而是切换到监督者模式，这样监督者模式就可以终止该应用程序，因为它做了不该做的事情。第一章中的图1.1说明了这种组织结构。一个应用程序只能执行用户模式的指令（例如，加数字等），被称为在用户空间运行，而处于监督者模式的软件也可以执行特权指令，被称为在内核空间运行。运行在内核空间（或监督者模式）的软件被称为内核。

一个想要调用内核功能的应用程序（例如，xv6中的`read`系统调用）必须过渡到内核；一个应用程序不能直接调用内核功能。CPU提供一个特殊的指令，将CPU从用户模式切换到监督者模式，并在内核指定的入口点进入内核(RISC-V为此提供了ecall指令)。一旦CPU切换到监督者模式，内核就可以验证系统调用的参数（例如，检查传递给系统调用的地址是否是应用程序内存的一部分），决定是否允许应用程序执行所请求的操作（例如，检查是否允许应用程序写入指定文件），然后决定拒绝或执行。重要的是，内核要控制进入到监督者模式的入口点；如果应用程序可以决定内核的入口点，那么恶意的应用程序就可以，例如，在跳过参数验证的地方进入内核。

### 2.3 内核组织

一个关键的设计问题是操作系统的哪一部分应该在监督者模式下运行。一种可能性是，整个操作系统驻留在内核中，所有系统调用都在监督者模式下运行。这种组织被称为宏内核。

在这种组织中，整个操作系统以完全的硬件权限运行。这种组织很方便，因为操作系统设计者不必决定操作系统的哪一部分不需要全硬件权限。此外，操作系统的不同部分也更容易合作。例如，操作系统可能有一个同时由文件系统和虚拟内存系统共享的缓存区。

宏内核的缺点是，操作系统不同部分之间的接口往往很复杂（正如我们将在本文的其余部分看到的那样），因此操作系统的开发者很容易犯错误。在一个宏内核中，错误是致命的，因为监督者模式的错误往往会导致内核失败。如果内核失效，计算机就会停止工作，因此所有的应用程序也会失效。计算机必须重新启动才能恢复运转。

为了减少内核出错的风险，操作系统设计者可以尽量减少在监督者模式下运行的操作系统代码数量，而在用户模式下执行操作系统的大部分。这种内核组织被称为微内核。

![image-20220919151641519](http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20220919151641519.png)

图2.1说明了这种微内核设计。在图中，文件系统作为一个用户级进程运行。作为进程运行的操作系统向外提供各项服务(`servers`)。为了允许应用程序与文件服务器进行交互，内核提供了一个进程间的通信机制，以便将消息从一个用户模式的进程发送到另一个进程。例如，如果像shell这样的应用程序想读或写一个文件，它就向文件服务(`file server`)发送一个消息并等待响应。

在微内核中，内核接口由一些低级函数组成，用于启动应用程序、发送消息、访问设备硬件等。这种组织方式使内核相对简单，因为大部分操作系统都驻留在用户级别。

在现实世界中，宏内核和微内核都很流行。许多Unix内核是宏内核。例如，Linux就是宏内核，但是有些操作系统是以用户级服务运行的（例如，windows系统）。Linux为操作系统密集型的应用提供了高性能，部分原因是内核的子系统可以紧密集成。

Minix、L4和QNX等操作系统被组织成带有服务(`server`)的微内核，并被广泛部署在嵌入式环境中。L4的一个变种，seL4，很小，并已经被验证了内存安全和其他安全属性\[8]。

哪种组织更好？操作系统的开发者之间有很多争论，而且没有结论。此外，这在很大程度上取决于 "更好 "的含义：更快的性能、更小的代码量、内核的可靠性、整个操作系统（包括用户级服务）的可靠性，等等。

还有一些实际的考虑，可能比哪个组织的问题更重要。有些操作系统是微内核，但出于性能的考虑，在内核空间运行一些用户级服务。有些操作系统是宏内核，因为它们就是这样开始的，而且没有什么动力去转向微内核组织，因为新的功能可能比重写现有的操作系统以适应微内核更重要。

从本书的角度来看，微内核和宏内核操作系统有许多相同的设计思路。他们实现系统调用，使用页表，处理中断，支持进程，使用锁进行并发控制，实现文件系统，等等。本书的重点是这些核心思想。

Xv6和大多数Unix操作系统一样，采用宏内核。因此，xv6的内核接口与操作系统接口相对应，在内核实现了完整的操作系统。由于xv6不提供很多服务，它的内核可能比一些微内核还要小，但从概念上讲xv6是宏内核。

### 2.4 代码: xv6组织结构

![image-20220919153329402](http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20220919153329402.png)

xv6内核的源代码在`kernel/`子目录下。源文件被分成若干个文件，遵循一个粗略的模块化概念；图2.2列出了这些文件。模块间的接口定义在defs.h（kernel/defs.h）

### 2.5 进程概述

xv6中隔离的基本单位（如同其他Unix操作系统）是进程。进程的抽象防止一个进程破坏或窥探另一个进程的内存、CPU、文件描述符等。还可以防止某个进程破坏内核，因此进程不能打破内核的隔离机制。内核必须谨慎地实现进程抽象，因为一个有缺陷的或恶意的应用程序可能会欺骗内核或硬件做一些坏事（例如，绕过隔离）。内核用来实现进程的机制包括用户/监督者模式标志、地址空间和线程的时间切片。

为了帮助实现隔离，进程抽象给程序提供了一个假象，即它自己独享机器。进程为程序提供了一个看似私有的内存系统或地址空间，其他进程无法读取或写入。进程还为程序提供了似乎是它独有的CPU来执行程序的指令。

Xv6使用页表(`page tables`)（由硬件实现）来给每个进程提供自己的地址空间。RISC-V页表将虚拟地址（指令操作的地址）翻译（或 "映射"）为物理地址（CPU发送至主内存的地址）。

![image-20220919153906893](http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20220919153906893.png)

Xv6为每个进程维护一个单独的页表，定义了该进程的地址空间。如图2.3所示，一个地址空间包括从虚拟地址0开始的进程的内存。首先是指令，其次是全局变量，然后是堆栈，最后是一个 "堆 "区（用于malloc），进程可以根据需要扩展。有一些因素限制了进程地址空间的最大尺寸：RISC V上的指针是64位宽；硬件在页表中查找虚拟地址时只使用低39位；而xv6只使用这39位中的38位。因此，最大的地址是$2^{38} - 1$ = 0x3fffffffff，也就是MAXVA(kernel/riscv.h:363)。在地址空间的顶部，Xv6为`trampoline`保留了一个页面，并为进程的`trapframe`保留了一个页面。Xv6使用这两个页面进入和返回内核；`trampoline`页面包含了进入和离开内核的代码，`trapframe`对于保存/恢复用户进程的状态是必要的，我们将在第四章中解释。

xv6内核为每个进程维护着许多状态，它将这些状态收集到`struct proc`(kernel/proc.h:85)中。一个进程最重要的内核状态是它的页表、内核栈和运行状态。我们将使用符号`p ->xxx`来指代proc结构中的元素；例如，p->pagetable\`是一个指向进程页表的指针。

每个进程都有一个执行线程（简称线程），执行该进程的指令。一个线程可以被暂停，恢复。为了在进程之间透明地切换，内核会暂停当前运行的线程，并恢复另一个进程的线程。一个线程的大部分状态（局部变量、函数调用返回地址）都存储在线程的堆栈中。每个进程有两个堆栈：一个用户堆栈和一个内核堆栈（p ->kstack）。当进程在执行用户指令时，只有它的用户堆栈在使用，而它的内核堆栈是空的。当进程进入内核时（为了系统调用或中断），内核代码在进程堆栈中执行；当进程在内核中时，其用户堆栈仍然包含保存的数据，但并不使用。一个进程的线程在用户栈和内核栈之间交替使用。内核堆栈是独立的（并且不受用户代码的保护），因此，即使进程破坏了它的用户堆栈，内核也可以执行。

一个进程可以通过执行RISC-V的`ecall指令`来进行系统调用。这条指令提高了硬件权限级别，并将程序计数器改为内核定义的入口点。入口点的代码切换到内核堆栈，执行实现系统调用的内核指令。当系统调用完成后，内核切换回用户堆栈，并通过调用`sret指令`返回到用户空间，该指令降低了硬件权限级别，并在系统调用指令后恢复执行用户指令。进程的线程可以在内核中 "阻塞 "以等待I/O，当I/O完成后再继续执行。

`p->state`表示进程是分配、准备运行、运行、等待I/O、还是退出。

`p->pagetable`持有进程的页表，格式是RISC-V所规定的。Xv6使分页硬件在用户级别执行进程时使用该进程的页表`p->pagetable`。一个进程的页表同时也是记录了所有分配给该进程的物理内存页。

总之，进程捆绑了两个设计理念：一个是地址空间，让进程有自己内存的错觉；另一个是线程，让进程有自己CPU的错觉。在xv6中，一个进程由一个地址空间和一个线程组成。在实际的操作系统中，一个进程可能有不止一个线程，以充分利用多个CPU。

### 2.6 代码：启动xv6，第一个进程和系统调用

为了使xv6更加具体，我们将概述内核如何启动和运行第一个进程。后面的章节将更详细地描述这个机制。

当RISC-V计算机开机时，它初始化自己并运行一个存储在只读存储器中的引导加载器。引导加载器将xv6内核加载到内存中。然后，在机器模式下，CPU从`_entry`(kernel/entry.S:7)开始执行xv6。RISC-V开始时禁用分页硬件：虚拟地址直接映射到物理地址。

装载器将xv6内核加载到物理地址0x80000000的内存中。之所以把内核放在0x80000000而不是0x0处，是因为地址范围\[0x0:0x80000000]是I/O设备地址。

`_entry`内的指令设置了一个栈，以便xv6可以运行C代码。xv6在`start.c` (kernel/start.c:11)中声明了一个栈`stack0`。`_entry`中的代码将栈指针sp初始化为`stack 0+4096`，即指向栈的顶部，因为RISC-V上的堆栈是向下增长的。现在内核有了一个堆栈，`_entry`在start中调用C代码(kernel/start.c:21)。

函数`start`执行一些只允许在机器模式下进行的配置，然后切换到监督者模式。为了进入监督者模式，RISC-V提供了指令`mret`，这个指令常常被用来从监管者模式返回机器模式，但start并不是从监管者模式中返回的，因此需要把自己设置成监管者模式，使得机器看起来是要从监管者模式进行返回：在寄存器mstatus中把之前的特权模式设置为监督者模式，通过把main的地址写入寄存器mepc来设置返回地址为main，通过在页表寄存器satp中写入0来禁止监督者模式下的虚拟地址转换，并把所有中断和异常委托给监督者模式。

在进入监督模式之前，`start`还执行了一项任务：对时钟芯片进行编程以产生时钟中断。在完成这些工作后，`start`通过调用`mret` "返回 "到监督者模式。之后让程序计数器指向`main`（kernel/main.c:11）。

在`main`（kernel/main.c:11）初始化了几个设备和子系统之后，它通过调用`userinit`(kernel/proc.c:233）创建了第一个进程。`initcode.S` (user/initcode.S:3) 将exec系统调用的编号`SYS_EXEC` (kernel/syscall.h:8)加载到寄存器a7，然后调用ecall重新进入内核。

内核在`syscall`（kernel/syscall.c:132）中使用寄存器a7中的数字来调用所需的系统调用。系统调用表（kernel/syscall.c:107）将SYS\_EXEC映射到`sys_exec`，之后内核调用它。正如我们在第一章中所看到的，exec用一个新的程序（在这里是`/init`）替换当前进程的内存和寄存器。

一旦内核完成了`exec`，它就返回到用户空间的`/init`进程中。`init`（user/init.c:15）在需要时创建一个新的控制台设备文件，将其作为文件描述符0、1和2打开。然后它在控制台启动一个shell。系统启动完成。

### 2.7 安全模式

你可能想知道操作系统是如何处理有漏洞的或恶意的代码的。因为处理恶意代码严格来说比处理意外的bug更难，所以把这个话题看作与安全有关是合理的。下面是操作系统设计中典型的安全假设和安全目标的概览。

操作系统必须假设一个进程的用户级代码会尽力破坏内核或其他进程。用户代码可能试图在其允许的地址空间之外解引用指针；它可能试图执行任何RISC-V指令，甚至那些不是为用户代码准备的指令；它可能试图读写任何RISC V控制寄存器；它可能试图直接访问设备硬件；它可能向系统调用传递巧妙的值，试图欺骗内核崩溃或做一些愚蠢的事情。内核的目标是限制每个用户进程，这样它所能做的就是读/写/执行自己的用户内存，使用32个通用的RISC-V寄存器，并以系统调用所允许的方式影响内核和其他进程。内核必须防止任何其他行动。这通常是内核设计中的一个绝对要求。

对内核自身代码的期望则是完全不同的。内核代码被认为是由善意和谨慎的程序员编写的。内核代码被期望是没有错误的，当然也不包含任何恶意的东西。这个假设影响了我们分析内核代码的方式。例如，有许多内部内核函数（例如，自旋锁），如果内核代码不正确地使用它们，会导致严重的问题。当检查任何特定的内核代码时，我们必须说服自己，它的行为是正确的。我们假设内核代码总体上是正确编写的，并且遵循所有关于使用内核自身函数和数据结构的规则。在硬件层面上，RISC-V的CPU、RAM、磁盘等被认为是按照文档中所宣传的那样运行，没有硬件错误。

当然，在现实生活中，事情并不是那么简单的。很难防止聪明的用户代码通过消耗内核保护的资源--磁盘空间、CPU时间、进程表槽等，使系统无法使用（或者导致系统恐慌）。通常不可能编写无缺陷的代码或设计无缺陷的硬件；如果恶意用户代码的编写者知道内核或硬件的缺陷，他们会利用它们。即使在成熟的、广泛使用的内核中，如Linux，人们也会不断发现新的漏洞。在内核中设计保障措施以防止它有bug的可能性是值得的：断言、类型检查、堆栈保护页等等。最后，用户和内核代码之间的区别有时是模糊的：一些有特权的用户级进程可能提供基本的服务，并有效地成为操作系统的一部分，在一些操作系统中，有特权的用户代码可以向内核插入新的代码（如Linux的可加载内核模块）。

### 2.8 现实世界

大多数操作系统都采用了进程的概念，而且大多数进程看起来与xv6的相似。然而，现代操作系统在一个进程中支持多个线程，以使一个进程能够利用多个CPU。支持一个进程中的多个线程涉及到xv6所没有的大量机制，包括潜在的接口变化（例如Linux的clone，fork的一个变种），以控制线程共享进程的方方面面。

2.9 练习

1. 给xv6增加一个系统调用，返回可用的内存量。
