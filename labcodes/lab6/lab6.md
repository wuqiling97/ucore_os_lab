# Lab6

## 知识点

本实验涉及到了以下课程上提到的知识点：

1. RR算法，Stride算法和多级反馈队列算法
2. 调度算法的准则

本实验涉及的其他知识点：

​  1. skew heap的概念

OS原理中很重要，但在实验中没有对应上的知识点：

1. 其他的调度算法
2. 实时调度，多处理器调度，优先级反置

## 练习0

在复制了lab1-5的代码之后，需要做一些修改，包括：

`trap.c`:

1. `trap_dispatch` 在每次时钟中断的时候调用sched_class_proc_tick函数，将current进程设置为需要调度

`proc.c`:

1. `alloc_proc` 需要设置新的成员变量，但是由于我使用memset，所以只需要增加初始化run_link的语句即可。

## 练习1

##### 请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程

各个函数指针及其功能如下：

- init：初始化调度器的成员变量
- enqueue：将被调度出去的进程放入等待队列中
- dequeue：从等待队列当中移出一个进程
- pick_next：选取下一个要运行的进程
- proc_tick：每次时钟中断的时候调用，更新进程相关信息。

ucore的调度分为两种：

- `do_exit`, `do_wait`, `init_main`, `cpu_idle`, `lock`函数中调用`schedule`函数为主动放弃使用权，因为这些操作耗时较长，调用`schedule`可以避免CPU资源浪费
- `trap`函数中，处于用户态且需要调度的进程会调用`schedule`。进程在需要调度前，每次时钟中断，就会作为`sched_class_proc_tick `的参数，将time_slice-1，如果time_slice降低为0，就会将need_resched设为1

在`schedule`被调用之后，sched_class当中的函数才开始发挥作用，以下以Round Robin算法为例。首先，如果当前进程处于RUNNABLE的状态，就会调用`enqueue`将current放入队列run_queue尾部；接着通过`pick_next`选出下一个要执行的进程next，若next不为空，就用`dequeue`将其从run_queue当中移出，为空就将next设置为idle线程。最后，调用`proc_run`让next运行。

##### 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

算法需要给每一个优先级维护一个队列，并维护一个当前应该选择的优先级状态。对于每一个进程，需要增加一个成员变量来记录优先级，初始化时，设置为最高优先级。每个函数修改为：

- init：初始化算法所需的变量
- enqueue：如果当前进程剩余时间片不为0，需要将优先级降低一级。随后将进程放入对应优先级的队列当中，并将时间片设置为该优先级应有的时间片。
- dequeue：将相应优先级队列中的该进程移出即可
- pick_next：根据调度算法维护当前应该选择的优先级。随后根据此优先级，从对应队列中选出进程。
- proc_tick：功能不变

## 练习2

##### 实现 Stride Scheduling 调度算法

Stride调度算法每次进行pick_next的时候都是选择stride最小的进程，因此最好使用堆来实现。ucore代码当中已经提供了skew_heap，因此使用它来实现。

将sched_class的各个接口修改为：

- init：增加`lab6_run_pool=NULL`。
- enqueue：将传入的进程proc放入堆当中，更新堆顶指针，并将proc_num+1。随后更新proc的时间片，并设置rq指针。
- dequeue：将指定的进程从堆当中删除，并将proc_num-1
- pick_next：选择stride最小的进程，也就是堆顶的进程，作为即将使用CPU的进程返回。此外，还需要将进程的stride增加`BIG_STRIDE / priority`
- proc_tick：与Round Robin算法一致。

因为调整堆的顺序的时候需要将stride相减，因此BIG_STRIDE最大值为int的最大值，即0x7FFFFFFF。实践中需要设置为priority的最大值，否则对于priority最大的进程`BIG_STRIDE / priority`将一直为0，该进程将一直被选中。

##### 与参考答案的区别

答案还实现了链表版本的Stride调度算法，但是我没有实现。此外，上文中提到的`BIG_STRIDE / priority == 0`的情况我额外做了判断，如果为0，那么让stride+1。





