## Locking

用户空间应用程序和内核自身都需要保护资源，特别是后者。在SMP系统上，各个CPU可能同时处于核心态，在
理论上可以操作所有现存的数据结构。为阻止CPU彼此干扰，需要通过锁保护内核的某些范围。锁可以确保每次
只能有一个CPU访问被保护的范围。

###　竞态条件

我们考虑系统通过两种接口从外部设备读取数据的情况。独立的数据包以不定间隔通过两个接口到达，保存在不同文件中。为记录数据包到达的次序，在文件名之后添加了一个号码，表明数据包的序号。通常的一系列文件名是act1.fil、act2.fil、act3.fil，等等。可使用一个独立的变量来简化两个进程的工作。该变量保存在由两个进程共享的内存页中，且指定了下一个未使用的序号（为简明起见，在下文我称该变量为counter）。
在一个数据包到达时，进程必须执行一些操作，才能正确地保存数据。
*（1）从接口读取数据。
*（2）用序号counter构造文件名，打开一个文件。
*（3）将序号加1。
*（4）将数据写入文件，然后关闭文件。
上述的软件系统会发生错误吗？如果每个进程都严格遵守上述过程，并在适当的位置对状态变量加1，那么上述过程不仅适用于两个进程，也可用于多个进程。事实上，大多数情况下上述过程都会正确运作，但在某些情况下会出错，而这也是分布式程序设计的真正困难所在。我们设个陷阱，分别将从接口读取数据的进程称作进程1和
进程2。我们给出的场景中，已经保存了若干文件，比如说总共有12文件。因此counter的值是13。显然是个“凶兆”……进程1从接口接收一个刚到达的新数据块。它忠实地用序号13构造文件名并打开一个文件，而同时调度器被激活并确认该进程已经消耗了足够的CPU时间，必须由另一个进程替换，并且假定是进程2。要注意，此时进程1读取了counter的值，但尚未对counter加1。在进程2开始运行后，同样从其对应的接口读取数据，并开始执行必要的操作以保存这些数据。它会读取counter的值，用序号13构造文件名打开文件，将counter加1，counter从13变为14。接下来它将数据写入文件，最后结束。不久，又轮到进程1再次运行。它从上次暂停处恢复执行，并将counter加1，counter从14变为15。接下来它将数据写入到用序号13打开的文件，当然，在这样做的时候，会覆盖进程2已经保存的数据。这简直是祸不单行，丢失了一个数据记录，而且序号14也变得不可用了。修改程序接收数据之后的处理步骤，可以防止该错误。举例来说，进程可以在读取counter的值之后立即将counter加1，然后再去打开文件。但再想想，问题远远不会这么简单。因为我们总是可以设计出一些导致致命错误的情形。

因此，我们很快就意识到了：如果在读取counter的值和对其加1之间发生调度，则仍然会产生不一致的情况。几个进程在访问资源时彼此干扰的情况通常称之为竞态条件（race condition）。在对分布式应用编程时，这种情况是一个主要的问题，因为竞态条件无法通过系统的试错法检测。相反，只有彻底研究源代码（深入了解各种可能发生的代码路径）并通过敏锐的直觉，才能找到并消除竞态条件。由于导致竞态条件的情况非常罕见，因此需要提出一个问题：是否值得做一些（有时候是大量的）工作来保护代码避免竞态条件。在某些环境中（飞机的控制系统、重要机械的监控、危险装备），竞态条件是致命问题。即使在日常软件项目中，避免潜在的竞态条件也能大大提高程序的质量以及用户的满意度。为改进Linux内核对多处理器的支持，我们需要精确定位内核中暗藏竞态条件的范围，并提供适当的防护。由于缺乏保护而导致的出乎意料的系统崩溃和莫名其妙的错误，这些都是不可接受的。

###　临界区

这个问题的本质是：进程的执行在不应该的地方被中断，从而导致进程工作得不正确。

显然，一种可能的解决方案是标记出相关的代码段，使之无法被调度器中断。尽管这种方法原则上是可行的，但有几个内在问题。在某种情况下，有问题的程序可能迷失在标记的代码段中无法退出，因而无法放弃CPU，进而导致计算机不可用。因此我们必须立即放弃这种解决方案。问题的解决方案不一定要求临界区是不能中断的。只要没有其他的进程进入临界区，那么在临界区中执行的进程完全是可以中断的。这种严格的禁止条件，可以确保几个进程不能同时改变共享的值，我们称为互斥（mutual exclusion）。也就是说，在给定时刻，只有一个进程可以进入临界区代码。有许多方法可以设计这种类别的互斥方法（不考虑技术实现问题）。但所有的设计都必须保证，无论在何种情况下都要确保排他原则（exclusion principle）。这种保证决不能依赖于所涉及处理器的数目或速度。如果存在这样的依赖（以至于解决方案只适用于特定硬件配置下的给定计算机系统），那么该方案将是不切实际的。因为它无法提供通用的保护机制，而这正是我们所需要的。进程不应该允许彼此阻塞或永久停止。尽管这里描述了一个可取的目标，但它并不总是能够用技术手段实现，读者从下文可以看到这一点。经常需要程序员未雨绸缪，以避免问题的发生。应用何种原理来支持互斥方法？在多任务和多用户系统的历史上，人们提出了许多不同的解决方案，但都各有利弊。一些解决方案只是纯理论的，而另一些则已经在各种操作系统中付诸实践了。

###　内核锁机制

内核可以不受限制地访问整个地址空间。在多处理器系统上（或类似地，在启用了内核抢占的单处理器系统上），这会引起一些问题。如果几个处理器同时处于核心态，则理论上它们可以同时访问同一个数据结构，这刚好造成了前一节讲述的问题。在第一个提供了SMP功能的内核版本中，该问题的解决方案非常简单，即每次只允许一个处理器处于核心态。因此，对数据未经协调的并行访问被自动排除了。令人遗憾的是，该方法因为效率不高，很快被废弃了。现在，内核使用了由锁组成的细粒度网络，来明确地保护各个数据结构。如果处理器A在操作数据结构X，则处理器B可以执行任何其他的内核操作，但不能操作X。内核为此提供了各种锁选项，分别优化不同的内核数据使用模式。

* 原子操作：这些是最简单的锁操作。它们保证简单的操作，诸如计数器加1之类，可以不中断地原子执行。即使操作由几个汇编语句组成，也可以保证。

* 自旋锁：这些是最常用的锁选项。它们用于短期保护某段代码，以防止其他处理器的访问。在内核等待自旋锁释放时，会重复检查是否能获取锁，而不会进入睡眠状态（忙等待）。当然，如果等待时间较长，则效率显然不高。

* 信号量：这些是用经典方法实现的。在等待信号量释放时，内核进入睡眠状态，直至被唤醒。唤醒后，内核才重新尝试获取信号量。**互斥量是信号量的特例，互斥量保护的临界区，每次只能有一个用户进入。**

* 读者/写者锁：这些锁会区分对数据结构的两种不同类型的访问。任意数目的处理器都可以对数据结构进行并发读访问，但只有一个处理器能进行写访问。事实上，在进行写访问时，读访问是无法进行的。

在使用自旋锁时必须要注意下面两点。

*（1）如果获得锁之后不释放，系统将变得不可用。所有的处理器（包括获得锁的在内），迟早需要进入锁对应的临界区。它们会进入无限循环等待锁释放，但等不到。这产生了死锁，从名称来看，这是个应该避免的状况。

*（2）自旋锁决不应该长期持有，因为所有等待锁释放的处理器都处于不可用状态，无法用于其他工作.

由自旋锁保护的代码不能进入睡眠状态。遵守该规则不像看上去那样简单：避免直接进入睡眠状态并不复杂，但还必须保证在自旋锁保护的代码所调用的函数也不会进入睡眠状态！一个特定的例子是kmalloc函数。通常将立刻返回请求的内存，但在内存短缺时，该函数可以进入睡眠状态。如果自旋锁保护的代码会分配内存，那么该代码大多数时间可能工作完全正常，但少数情况下会造成失败。当然，这种问题很难重现和调试。因此应该非常注意在自旋锁保护的代码中调用的函数，确保这些函数在任何情况下都不会睡眠。

在单处理器系统上，自旋锁定义为空操作，因为不存在几个CPU同时进入临界区的情况。但如果启用了内核抢占，这种说法就不适用了。如果内核在临界区中被中断，而此时另一个进程进入临界区，这与SMP系统上两个处理器同时在临界区执行的情况是等效的。通过一个简单的技巧可以防止这种情况发生：内核进入到由自旋锁保护的临界区时，就停用内核抢占。在启用了内核抢占的单处理器内核中，spin_lock（基本上）等价于pre-empt_disable，而spin_unlock则等价于preempt_enable。

自旋锁当前的持有者无法多次获得同一自旋锁！在函数调用了其他函数，而这些函数每次都操作同一个锁时，这种约束特别重要。如果已经获得一个锁，而调用的某个函数试图再次获得该锁，尽管当前的代码路径已经持有该锁，也同样会发生死锁——处理器等待自身释放持有的锁，这就有得等了……