# Pintos Project1：Thread 实验报告



## 一、实验简介

​		Pintos是Standford大学为操作系统课程专门开发的一个基于80x86架构的简单操作系统框架(A simple operating system framework)。他支持内核线程、加载和运行用户程序以及文件系统。

## 二、Mission1：重新实现timer_sleep函数

### 2.1 任务介绍

​		修改pintos的线程休眠函数来保证pintos不会在一个线程休眠时忙等待。

### 2.2 实验分析

#### 		2.2.1 存储线程信息的数据结构

​		thread.h定义了一个thread的结构体用于存储线程的信息，相当于进程控制块。

```c
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */
  };
```

#### 		2.2.2 timer_sleep函数

​		未修改之前的timer_sleep就是在ticks时间内， 如果线程处于running状态就不断把他扔到就绪队列不让他执行。

```c
void timer_sleep (int64_t ticks) 
{
  int64_t start = timer_ticks ();

  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks) 
    thread_yield ();
}
```

#### 		2.2.3 thread_yield函数

​		thread_yield其实就是把当前线程扔到就绪队列里， 然后重新schedule， 注意这里如果ready队列为空的话当前线程会继续在cpu执行。这里只是把线程放进调度队列，然后切换线程，但这时候休眠线程状态还是是ready，在一个tick内依然有可能会被调度，继续消耗cpu时间。造成了忙等待。

```c
void thread_yield (void) 
{
  struct thread *cur = thread_current ();
  enum intr_level old_level;
  
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread) 
    list_push_back (&ready_list, &cur->elem);
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}
```

#### 		2.2.4 schedule函数

​		schedule()是专门负责线程切换的函数，执行了以后会把当前线程放进队列里并调度出下一个线程

```c
static void schedule (void) 
{
  struct thread *cur = running_thread ();
  struct thread *next = next_thread_to_run ();
  struct thread *prev = NULL;

  ASSERT (intr_get_level () == INTR_OFF);
  ASSERT (cur->status != THREAD_RUNNING);
  ASSERT (is_thread (next));

  if (cur != next)
    prev = switch_threads (cur, next);
  thread_schedule_tail (prev);
}
```

#### 		2.2.5 总结

​		线程依然不断在cpu就绪队列和running队列之间来回， 占用了cpu资源， 这并不是我们想要的， 我们希望用一种唤醒机制来实现这个函数。 

### 2.3 具体实现

​		实现思路： 调用timer_sleep的时候直接把线程阻塞掉，然后给线程结构体加一个成员blocked_ticks来记录该线程被阻塞的时间 ， 然后利用操作系统自身的时钟中断（每个tick会执行一次）加入对线程状态的检测， 每次检测将blocked_ticks减1, 如果减到0就唤醒这个线程。

```c
void timer_sleep (int64_t ticks) 
{
    if (ticks <= 0) return;
    int64_t start = timer_ticks ();

    ASSERT (intr_get_level () == INTR_ON);
    enum intr_level old_level = intr_disable();
    struct thread* cur_thread = thread_current();
    cur_thread->blocked_ticks = ticks;
    thread_block();
    intr_set_level(old_level);
}

```
这里调用thread_block()函数，主要实现功能为设置进程的阻塞状态，并调用进程切换

给线程的结构体加上我们的blocked_ticks成员，用于记录该线程被阻塞的时间

```c
/* Record the time the thread has been blocked. */
int64_t blocked_ticks;
```

然后利用操作系统自身的时钟中断(timer_interrupt）加入对线程状态的检测， 每次检测将blocked_ticks减1, 如果减到0就唤醒这个线程。所以我们需要在线程被创建的时候初始化blocked_ticks为0， 在thread_create函数内添加如下代码：

```c
t->blocked_ticks = 0;
```

修改时钟中断处理函数， 加入线程sleep时间的检测， 检测哪些进程使用block状态并且含有剩余的休眠时间。加在timer_interrupt内。

```c
thread_foreach (check_blocked_thread, NULL);
```

check_blocked_thread函数的作用是，当前待检进程为阻塞状态并且阻塞时间剩余大于0个计时器刻度时执行，剩余刻度自减。若减为0，调用thread_unblock函数将当前进程加入到就绪队列。

```c
void check_blocked_thread(struct thread* t, void* aux UNUSED)
{
    if (t->status == THREAD_BLOCKED && t->blocked_ticks > 0)
    {
        t->blocked_ticks--;
        if (t->blocked_ticks == 0) thread_unblock(t);
    }
}
```

至此， timer_sleep函数唤醒机制就实现了



## 三、Mission 2：在Pintos中实现优先级调度

### 3.1 任务介绍

pintos project1 mission2的主要任务是在Pintos中实现优先级调度

### 3.2 实验分析

#### 3.2.1 线程的priority

在结构体thread里，线程成员本身就有一个priority

```c
int priority;                       /* Priority. */
```

priority的约束

```c
/* Thread priorities. */
#define PRI_MIN 0                       /* Lowest priority. */
#define PRI_DEFAULT 31                  /* Default priority. */
#define PRI_MAX 63                      /* Highest priority. */
```

#### 3.2.2 优先级调度

优先级调度的核心思想就是： 维持就绪队列为一个优先级队列。那么我们在什么时候会把一个线程丢到就绪队列中呢?

+ thread_unblock，即线程从阻塞态转为就绪态   
+ init_thread，即新建了一个线程
+ thread_yield，即把当前线程扔到就绪队列

也就是说，要保证每次将线程放入就绪队列后，就绪队列仍然是优先级队列

但是在上面三个函数里，将线程放入就绪队列时都是通过函数list_push_back来实现的，而list_push_back的作用是直接将线程放入队列的末尾，这显然与优先级队列的思想不符

#### 3.2.3 抢占式调度

抢占式调度的测试主要在priority-change与priority-preempt两个test中

会发生抢占式调度的情况有两个：

+ 在设置一个线程优先级要立即重新考虑所有线程执行顺序， 重新安排执行顺序
+ 在创建线程的时候， 新创建的线程比主线程优先级高

解决方法很简单，就是在发生上述两种情况时立即调用thread_yield

#### 3.2.4 优先级捐赠

优先级捐赠主要是为了解决，高优先级的任务因为低优先级任务占用资源而阻塞。在这时就需要将低优先级任务的优先级提升到等待它所占有的资源的最高优先级任务的优先级。

通过分析priority-donate-*系列的相关测试，可以得到以下逻辑

+ 在一个线程获取一个锁的时候， 如果拥有这个锁的线程优先级比自己低就提高它的优先级，并且如果这个锁还被别的锁锁着， 将会递归地捐赠优先级， 然后在这个线程释放掉这个锁之后恢复未捐赠逻辑下的优先级。
+ 如果一个线程被多个线程捐赠， 维持当前优先级为捐赠优先级中的最大值（acquire和release之时）。
+ 在对一个线程进行优先级设置的时候， 如果这个线程处于被捐赠状态， 则对original_priority进行设置， 然后如果设置的优先级大于当前优先级， 则改变当前优先级， 否则在捐赠状态取消的时候恢复original_priority。
+ 在释放锁对一个锁优先级有改变的时候应考虑其余被捐赠优先级和当前优先级。
+ 将信号量的等待队列实现为优先级队列。
+ 将condition的waiters队列实现为优先级队列。
+ 释放锁的时候若优先级改变则可以发生抢占。


### 3.3 具体实现

#### 3.3.1 实现优先级队列

根据上面的分析，实现优先级队列的第一步先要替换thread_unblock、init_thread、thread_yield里的list_push_back

将thread_unblock里的list_push_back替换为如下代码

```c
list_insert_ordered(&ready_list, &t->elem, (list_less_func*)&thread_cmp, NULL);
```

将init_thread里的list_push_back替换为如下代码

```c
list_insert_ordered(&all_list, &t->allelem, (list_less_func*)&thread_cmp, NULL);
```

将thread_yield里的list_push_back替换为如下代码

```
list_insert_ordered(&ready_list, &cur->elem, (list_less_func*)&thread_cmp, NULL);
```

其中list_insert_ordered函数是pintos已经实现好的，其作用描述如下：

*Inserts ELEM in the proper position in LIST, which must be sorted according to LESS given auxiliary data AUX. Runs in O(n) average case in the number of elements in LIST*

比较函数thread_cmp的实现如下

```c
bool thread_cmp(const struct list_elem* x, const struct list_elem* y, void* aux UNUSED)
{
    return list_entry(x, struct thread, elem)->priority > list_entry(y, struct thread, elem)->priority;
}
```

然后把condition的队列改成优先级队列

先修改cond_signal函数：

```c
void cond_signal (struct condition *cond, struct lock *lock UNUSED) 
{
  ASSERT (cond != NULL);
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (lock_held_by_current_thread (lock));

  if (!list_empty(&cond->waiters))
  {
      list_sort(&cond->waiters, cmp_priority_for_cond, NULL);
      sema_up(&list_entry(list_pop_front(&cond->waiters),struct semaphore_elem, elem)->semaphore);
  } 
}
```

其中比较函数cmp_priority_for_cond实现如下：

```c
bool cmp_priority_for_cond(const struct list_elem* x, const struct list_elem* y, void* aux UNUSED)
{
    struct semaphore_elem* sem_x = list_entry(x, struct semaphore_elem, elem);
    struct semaphore_elem* sem_y = list_entry(y, struct semaphore_elem, elem);
    return list_entry(list_front(&sem_x->semaphore.waiters), struct thread, elem)->priority > list_entry(list_front(&sem_y->semaphore.waiters), struct thread, elem)->priority;
}
```

然后把信号量的等待队列实现为优先级队列

修改sema_up:

```c
void sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty(&sema->waiters))
  {
      /*struct list_elem *first_priority_thread = list_max(&sema->waiters, thread_cmp, NULL);
      list_remove(first_priority_thread);
      thread_unblock(list_entry(first_priority_thread, struct thread, elem));*/
      list_sort(&sema->waiters, thread_cmp, NULL);
      thread_unblock(list_entry(list_pop_front(&sema->waiters), struct thread, elem));
  }
  sema->value++;
  thread_yield();
  intr_set_level (old_level);
}
```

修改sema_down:

```c
void sema_down (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  while (sema->value == 0) 
    {
      list_insert_ordered (&sema->waiters, &thread_current ()->elem, thread_cmp, NULL);
      thread_block ();
    }
  sema->value--;
  intr_set_level (old_level);
}
```

#### 3.3.2 实现抢占式调度

在线程设置优先级的时候调用thread_yield

```c
void thread_set_priority (int new_priority)
{
  thread_current ()->priority = new_priority;
  thread_yield ();
}
```

在创建线程的时候， 如果新创建的线程比主线程优先级高的话调用thread_yield

在thread_create最后把创建的线程unblock了之后加上如下代码

```c
if (thread_current()->priority < priority) thread_yield();
```

#### 3.3.3 实现优先级捐赠

修改thread数据结构， 加入以下成员：

```c
int original_priority;       //原始优先级
struct list hold_locks;      //该线程拥有的锁
struct lock *wait_for_lock;  //该线程所等待的锁
```

修改lock数据结构，加入以下成员：

```c
struct list_elem node;
int max_priority; 
```

修改lock_acquire函数

在P操作之前递归地实现优先级捐赠， 然后在被唤醒之后成为这个锁的拥有者。

```c
void
lock_acquire (struct lock *lock)
{
  struct thread *cur = thread_current ();
  struct lock *lk;

  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if (lock->holder != NULL && !thread_mlfqs)
  {
    cur->wait_for_lock = lock;
    lk = lock;
    while (lk && cur->priority > lk->max_priority)
    {
       lk->max_priority = cur->priority;
       donate_priority (lk->holder);
       lk = lk->holder->wait_for_lock;
    }
  }

  sema_down (&lock->semaphore);

  enum intr_level old_level  = intr_disable ();
  cur = thread_current ();
  if (!thread_mlfqs){
    cur->wait_for_lock = NULL;
    lock->max_priority = cur->priority;
    hold_lock (lock);
  }
  lock->holder = thread_current ();

  intr_set_level (old_level);
}
```

这里donate_priority和hold_lock封装成函数

```c
void donate_priority (struct thread *t)
{
   enum intr_level old_level = intr_disable ();
   renew_priority (t);
   if (t->status == THREAD_READY)
   {
    list_remove (&t->elem);
    list_insert_ordered (&ready_list, &t->elem, thread_cmp, NULL);
   }
   intr_set_level (old_level);
}

void hold_lock(struct lock *lock)
{
  enum intr_level old_level = intr_disable ();
  list_insert_ordered (&thread_current ()->hold_locks, &lock->node, lock_cmp, NULL);

  if (lock->max_priority > thread_current ()->priority)
  {
     thread_current ()->priority = lock->max_priority;  thread_yield ();
  }
  intr_set_level (old_level);
}

```

锁队列排序函数lock_cmp:

```c
bool lock_cmp(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)
{
  return list_entry (a, struct lock, node)->max_priority > list_entry (b, struct lock, node)->max_priority;
}
```

在lock_release函数加入以下语句:

```c
if (!thread_mlfqs)
    remove_lock (lock);
```

remove_lock实现如下:

```c
void remove_lock (struct lock *lock)
{
  enum intr_level old_level = intr_disable ();
  list_remove (&lock->node);
  renew_priority (thread_current ());
  intr_set_level (old_level);
}
```

当释放掉一个锁的时候， 当前线程的优先级可能发生变化， 用update_priority来处理。

如果这个线程还有锁， 就先获取这个线程拥有锁的最大优先级（可能被更高级线程捐赠）， 然后如果这个优先级比base_priority大的话更新的应该是被捐赠的优先级。

```c
void update_priority (struct thread *t,void *aux UNUSED)
{
  if (t == idle_thread)
    return;

  ASSERT (thread_mlfqs);
  ASSERT (t != idle_thread);

  t->priority = (PRI_MAX*fp_one - (t->recent_cpu / 4) - t->nice*2*fp_one) / fp_one;
  if(t->priority > PRI_MAX) t->priority = PRI_MAX;
  if(t->priority < PRI_MIN) t->priority = PRI_MIN;
}
```

然后在init_thread中加入初始化：

```c
t->original_priority = priority;
list_init (&t->hold_locks);
t->wait_for_lock = NULL;
```

修改thread_set_priority:

```c
void thread_set_priority (int new_priority) 
{
  if (thread_mlfqs) return;

  enum intr_level old_level = intr_disable ();

  struct thread *cur = thread_current ();
  int old_priority = cur->priority;
  cur->original_priority = new_priority;

  if (list_empty (&cur->hold_locks) || new_priority > old_priority)
  {
    thread_current ()->priority = new_priority;
    thread_yield();
  }
  intr_set_level (old_level);
}
```

至此,实现了优先级捐赠的逻辑。

## 四、Mission 3：实现多级反馈调度

### 4.1 任务介绍

这种类型的调度程序维护着几个准备运行的线程队列，其中每个队列包含具有不同优先级的线程。在任何给定时间，调度程序都会从优先级最高的非空队列中选择一个线程。如果最高优先级队列包含多个线程，则它们以“轮转(round robin)”顺序运行。

### 4.2 实验分析

#### 4.2.1 Niceness

每个线程还有一个整数*nice*值，该值确定该线程与其他线程应该有多“nice”。nice为零不会影响线程优先级。正的nice（最大值为20）会降低线程的优先级，并导致该线程放弃原本可以接收的CPU时间。另一方面，负的nice，最小为-20，往往会占用其他线程的CPU时间。初始线程以nice值为零开始。其他线程以从其父线程继承的nice值开始。

必须实现下面描述的功能，供测试程序使用

+ Function: int **thread_get_nice** (void) : 返回当前线程的nice值。
+ Function: void **thread_set_nice** (int new_nice) : 将当前线程的nice值设置为new_nice，并根据新值重新计算线程的优先级。如果正在运行的线程不再具有最高优先级，则让出。

#### 4.2.2 Calculating Priority

我们的调度程序具有64个优先级，因此有64个就绪队列，编号为0（`PRI_MIN`）到63（`PRI_MAX`）。较低的数字对应较低的优先级，因此优先级0是最低优先级，优先级63是最高优先级。线程优先级最初是在线程初始化时计算的。每个线程的第四个时钟滴答也会重新计算一次。无论哪种情况，均由以下公式确定 : ==priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)==

其中recent_cpu是线程最近使用的CPU时间的估计值，而nice是线程的nice值

该公式把最近获得CPU时间的线程赋予较低的优先级，以便在下次调度程序运行时重新分配CPU。这是防止饥饿的关键：最近未获得任何CPU时间的线程的recent_cpu为0，除非高nice值，否则它很快会获得CPU时间。

#### 4.2.3 计算 recent_cpu

我们希望recent_cpu测量每个进程“最近”获得了多少CPU时间。recent_cpu的初始值在创建的第一个线程中为0，在其他新线程中为父级的值。每次发生计时器中断时，除非正在运行的空闲线程，否则recent_cpu仅对正在运行的线程加1。另外，每秒使用以下公式重新计算每个线程（无论处于运行、就绪还是阻塞）的recent_cpu的值：

==recent_cpu = (2*load_avg)/(2*load_avg + 1) * recent_cpu + nice==

其中load_avg是准备运行的线程数的移动平均值。如果load_avg为1，表示平均一个线程正在争夺CPU。

#### 4.2.4 计算 load_avg

load_avg（通常称为系统平均负载）估计过去一分钟准备运行的线程的平均值。与recent_cpu一样，它是指数加权的移动平均值。与priority和recent_cpu不同，load_avg是系统级的，而不是特定于线程的。在系统引导时，它将初始化为0。此后每秒一次，将根据以下公式进行更新：==load_avg = (59/60)*load_avg + (1/60)*ready_threads==

其中ready_threads是在更新时正在运行或准备运行的线程数（不包括空闲线程）。

#### 4.2.5 浮点数运算

上述计算又涉及到了浮点数运算的问题， pintos本身并没有实现这个， 需要自己来实现

The following table summarizes how fixed-point arithmetic operations can be implemented in C. In the table, `x` and `y` are fixed-point numbers, `n` is an integer, fixed-point numbers are in signed p.q format where p + q = 31, and `f` is `1 << q：` 

|          **Convert `n` to fixed point:**           | **n * f**                                                    |
| :------------------------------------------------: | ------------------------------------------------------------ |
| **Convert `x` to integer (rounding toward zero):** | **x / f**                                                    |
| **Convert `x` to integer (rounding to nearest):**  | **`(x + f / 2) / f` if `x >= 0`, <br/>`(x - f / 2) / f` if `x <= 0`.** |
|                **Add `x` and `y`:**                | **`x + y`**                                                  |
|             **Subtract `y` from `x`:**             | **`x - y`**                                                  |
|                **Add `x` and `n`:**                | **`x + n * f`**                                              |
|             **Subtract `n` from `x`:**             | **`x - n * f`**                                              |
|              **Multiply `x` by `y`:**              | **`((int64_t) x) * y / f`**                                  |
|              **Multiply `x` by `n`:**              | **`x * n`**                                                  |
|               **Divide `x` by `y`:**               | **`((int64_t) x) * f / y`**                                  |
|               **Divide `x` by `n`:**               | **`x / n`**                                                  |



#### 4.2.6 总结

以下公式总结了实现调度程序所需的计算。它们不是调度程序要求的完整描述。

每个线程直接在其控制下的nice值在-20和20之间。每个线程还有一个优先级，介于0（`PRI_MIN`）到63（`PRI_MAX`）之间，每四个刻度使用以下公式重新计算一次：

`priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)`.

recent_cpu测量线程“最近”接收到的CPU时间。在每个计时器滴答中，运行线程的recent_cpu递增1。每秒一次，每个线程的recent_cpu都以这种方式更新：

`recent_cpu = (2*load_avg)/(2*load_avg + 1) * recent_cpu + nice`.

load_avg估计过去一分钟准备运行的平均线程数。它在引导时初始化为0，并每秒重新计算一次，如下所示：

`load_avg = (59/60)*load_avg + (1/60)*ready_threads`.

其中ready_threads是在更新时正在运行或准备运行的线程数（不包括空闲线程）。

### 4.3 具体实现

实现思路： 在timer_interrupt中固定一段时间计算更新线程的优先级，这里是每TIMER_FREQ时间更新一次系统load_avg和所有线程的recent_cpu， 每4个timer_ticks更新一次线程优先级， 每个timer_tick running线程的recent_cpu加1

在fixed_point.h中实现浮点运算逻辑：

```c
#ifndef __THREAD_FIXED_POINT_H
#define __THREAD_FIXED_POINT_H

/* Basic definitions of fixed point. */
typedef int fixed_t;
/* 16 LSB used for fractional part. */
#define FP_SHIFT_AMOUNT 16
/* Convert a value to fixed-point value. */
#define FP_CONST(A) ((fixed_t)(A << FP_SHIFT_AMOUNT))
/* Add two fixed-point value. */
#define FP_ADD(A,B) (A + B)
/* Add a fixed-point value A and an int value B. */
#define FP_ADD_MIX(A,B) (A + (B << FP_SHIFT_AMOUNT))
/* Substract two fixed-point value. */
#define FP_SUB(A,B) (A - B)
/* Substract an int value B from a fixed-point value A */
#define FP_SUB_MIX(A,B) (A - (B << FP_SHIFT_AMOUNT))
/* Multiply a fixed-point value A by an int value B. */
#define FP_MULT_MIX(A,B) (A * B)
/* Divide a fixed-point value A by an int value B. */
#define FP_DIV_MIX(A,B) (A / B)
/* Multiply two fixed-point value. */
#define FP_MULT(A,B) ((fixed_t)(((int64_t) A) * B >> FP_SHIFT_AMOUNT))
/* Divide two fixed-point value. */
#define FP_DIV(A,B) ((fixed_t)((((int64_t) A) << FP_SHIFT_AMOUNT) / B))
/* Get integer part of a fixed-point value. */
#define FP_INT_PART(A) (A >> FP_SHIFT_AMOUNT)
/* Get rounded integer of a fixed-point value. */
#define FP_ROUND(A) (A >= 0 ? ((A + (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT) \
        : ((A - (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT))

#endif /* thread/fixed_point.h */
```

 thread结构体加入以下成员：

```c
int nice;     
int recent_cpu;
```

在线程初始化的时候初始化这两个新的成员, 在init_thread中加入以下代码：

```c
t->nice = 0;
t->recent_cpu = 0;
```

在thread.c中加入全局变量

```c
fixed_t load_avg;
```

并在thread_start中初始化：

```c
load_avg = FP_CONST (0);
```

修改timer_interrup，加入以下代码： 

```c
if (thread_mlfqs)
  {
    increase_recent_cpu();
    if (ticks % TIMER_FREQ == 0)
    {
      update_load_avg ();
      thread_foreach(&update_recent_cpu,NULL);
    }
    else if (ticks % 4 == 0)  thread_foreach(&update_priority,NULL);
  }
```

其中increase_recent_cpu、update_load_avg、update_recent_cpu、update_priority函数的实现如下

```c
void increase_recent_cpu(void)
{
  ASSERT (thread_mlfqs);
  ASSERT (intr_context ());
  struct thread *cur = thread_current ();
  if (cur != idle_thread) cur->recent_cpu = cur->recent_cpu + fp_one;
}

void update_load_avg(void)
{
  int size=list_size(&ready_list);
  if(thread_current()!=idle_thread)
    size++;
  load_avg = load_avg*59/60 + (size)*fp_one/60;
}

void update_recent_cpu(struct thread *t,void *aux UNUSED)
{
  if(t==idle_thread) return;
  t->recent_cpu = (((int64_t)(2*load_avg))*fp_one/(2*load_avg+fp_one))*t->recent_cpu/fp_one+t->nice*fp_one;
}

void update_priority (struct thread *t,void *aux UNUSED)
{
  if (t == idle_thread)
    return;

  ASSERT (thread_mlfqs);
  ASSERT (t != idle_thread);

  t->priority = (PRI_MAX*fp_one - (t->recent_cpu / 4) - t->nice*2*fp_one) / fp_one;
  if(t->priority > PRI_MAX) t->priority = PRI_MAX;
  if(t->priority < PRI_MIN) t->priority = PRI_MIN;
}
```

最后完成任务要我们实现的四个函数：

```c
void thread_set_nice (int nice UNUSED) 
{
  struct thread *t = thread_current();
  t->nice = nice;
  update_priority(t,NULL);
  thread_yield();

}

/* Returns the current thread's nice value. */
int thread_get_nice (void) 
{
  return thread_current()->nice;
  return 0;
}

/* Returns 100 times the system load average. */
int thread_get_load_avg (void) 
{
  int avg = load_avg;
  if(avg>=0)
    avg = (avg+fp_one/2)/fp_one;
  else
    avg = (avg-fp_one/2)/fp_one;
  return avg*100;
}

/* Returns 100 times the current thread's recent_cpu value. */
int thread_get_recent_cpu (void) 
{
  int cpu = thread_current()->recent_cpu;
  if(cpu>=0)
    cpu = (cpu+fp_one/2)/fp_one;
  else
    cpu = (cpu-fp_one/2)/fp_one;
  return cpu*100;
}
```

## 五、实验心得

pintos这个项目让我们脱离了书本上刻板的知识，通过实践更好地理解OS这门课中的许多概念。比如项目中涉及到“优先级调度”、“信号量”、“锁”等等知识点，它们在pintos中串成了一个整体出现，这些概念都以看得见、摸得着的c代码的形式展现于眼前，是一个理解OS非常好的途径。虽然实验难度较大，但是收获很多。