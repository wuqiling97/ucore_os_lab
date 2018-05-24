# Lab7

## 知识点

本实验涉及到了以下课程上提到的知识点：

1. 禁用中断的同步方法
2. 信号量
3. 条件变量和管程
4. 哲学家就餐问题

本实验涉及的其他知识点：

   无

OS原理中很重要，但在实验中没有对应上的知识点：

1. 软件实现的同步方法
2. 读者-写着问题
3. 基于硬件指令（如test and set）的同步方法

## 练习0

在复制了lab1-6的代码之后，需要做一些修改，包括：

`trap.c`:

将`sched_class_proc_tick(current)`改为`run_timer_list()`

## 练习1

#### 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

##### 信号量的实现

信号量的实现中，定义了结构体`semaphore_t`，用来保存信号值value和等待队列wait_queue。对信号量有如下操作：

`sem_init`: 初始化value为对应的值，并且初始化等待队列

`down`: 对应课堂上讲的P()操作。具体实现和注释如下：

```c
bool intr_flag;
// 关中断来保证原子性
local_intr_save(intr_flag);
// 首先判断value是否大于0，是的话就减1并返回
if (sem->value > 0) {
    sem->value --;
    local_intr_restore(intr_flag);
    return 0;
}
// 否则将当前线程放入等待队列
wait_t __wait, *wait = &__wait;
wait_current_set(&(sem->wait_queue), wait, wait_state);
local_intr_restore(intr_flag);
// 然后进行调度
schedule();

// 调度回来
local_intr_save(intr_flag);
// 将自己从等待队列中移出
wait_current_del(&(sem->wait_queue), wait);
local_intr_restore(intr_flag);
// 等待flag未变化，说明获得了信号量，否则是由于其他原因唤醒的
if (wait->wakeup_flags != wait_state) {
    return wait->wakeup_flags;
}
return 0;
```

`up`: 对应V()操作。

```c
bool intr_flag;
// 关中断来保证原子性
local_intr_save(intr_flag);
{
    wait_t *wait;
    // 没有进程在等待队列中，直接将value+1
    if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
        sem->value ++;
    }
    // 否则唤醒等待队列中的第一个进程
    else {
        assert(wait->proc->wait_state == wait_state);
        wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
    }
}
local_intr_restore(intr_flag);
```

##### 哲学家就餐问题

lab7的代码不同于课堂上讲的任何一种方式，但是也能解决问题。代码定义了mutex用于保护临界区，以及s[5]代表哲学家是否能够开始就餐。

哲学家想要就餐的时候，会调用`phi_take_forks_sema `，在函数当中，首先记录该哲学家饥饿，然后进行`phi_test_sema `。`phi_test_sema `用于检查哲学家是否能开始就餐，如果能，那么将s[i]设置为1，并返回，否则直接返回。回到上一层函数，如果`phi_test_sema `将s[i]设为1，那么`down`就无需等待，相当于开始就餐；否则会将当前进程加入等待列表。

随后调用`phi_put_forks_sema `表示结束就餐，并对左右邻居调用`phi_test_sema `，若能够就餐，那么`up`会将由于`down`阻塞邻居唤醒，使他们开始就餐。

#### 给用户态进程/线程提供条件变量机制的设计方案 

可以增加一些系统调用，分别包装以下函数：

- sem_init: 初始化一个信号量，参数为value的初始值，并返回信号量的编号（出于安全考虑，不能返回指针）
- down: 参数为信号量id，相当于P操作，可能进入阻塞状态
- up: 参数为信号量id，相当于V操作
- 新增一个函数用于释放信号量

## 练习2

#### 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

内核态的管程机制通过Hoare管程实现，每次进入和退出的时候都需要执行

```c
//进入
down(&(mtp->mutex));
//退出
if(mtp->next_count>0)
    up(&(mtp->next));
else
    up(&(mtp->mutex));
```

相当于课堂上讲的lock->acquire(), lock->release()。

monitor的结构体如下

```c
typedef struct monitor{
    semaphore_t mutex;//管程锁      
    semaphore_t next; //记录阻塞的signal进程
    int next_count;   
    condvar_t *cv;    
} monitor_t;
```

条件变量的两个函数wait, signal的注释如下

```c
void cond_signal (condvar_t *cvp) {
    //如果有进程在等待条件变量
    if(cvp->count > 0) {
        //将自己放在next当中, 保证进程执行完毕或wait的时候能切回自己
        cvp->owner->next_count++;
        //唤醒等待进程
        up(&cvp->sem);
        down(&cvp->owner->next);
        cvp->owner->next_count--;
    }
}

void cond_wait (condvar_t *cvp) {
    //等待队列长度+1
    cvp->count++;
    //如果有阻塞在signal的进程, 则优先唤醒, 否则解除mutex锁
    if(cvp->owner->next_count > 0)
        up(&cvp->owner->next);
    else
        up(&cvp->owner->mutex);
    //请求条件变量
    down(&cvp->sem);
    cvp->count--;
}
```

##### 与参考答案的区别

因为伪代码已经很详细了，所以逻辑上完全一致

#### 用户态进程/线程提供条件变量机制的设计方案 

由于signal, wait操作都只需要调用信号量的接口，不需要其他的内核态特权，因此可以通过调用练习1当中提到的信号量的系统调用来实现用户带的条件变量。

#### 能否不用基于信号量机制来完成条件变量 

可以直接模仿课堂的设计来实现，但是需要锁机制的支持。假设有一个锁的结构体`Lock`支持`acquire, release`操作，那么可以如下实现：

```c++
class Condition {
    int numWait = 0;
    WaitQueue q;
    void wait(Lock& lock) {
        numWait++;
        q.add(current_thread);
        lock.release();
        schedule();
        lock.require();
    }
    void signal(Lock& lock) {
        if(numWait > 0) {
            t = q.head();
            q.remove(t);
            wakeup(t);
            numWait--;
        }
    }
};
```



