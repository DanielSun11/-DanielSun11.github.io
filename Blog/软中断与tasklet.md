# 一、软中断

## 1.1 何为软中断？

​	Linux 系统**为了解决中断处理程序执行过长的问题，将中断过程分成了两个阶段，分别是「上半部（Top Half）和下半部分（Bottom Half）」**。

- **上半部用来快速处理中断**。一般会暂时关闭中断请求，主要负责处理跟硬件紧密相关或者时间敏感的事情。

- **下半部用来延迟处理上半部未完成的工作**。下半部由中断上半部分触发，主要是负责上半部未完成的工作，通常都是耗时比较长的事情，特点是延迟执行，一般以「内核线程」的方式运行。

  软中断是内核实现中断下半部处理的机制之一。由中断上半部分触发，在专用的软中断守护线程ksoftirqd中或者在满足一定条件时在硬中断返回前被处理。

## 1.2 内核中记录软中断的数据结构

**软中断描述符**

```c
struct softirq_action
{
    //只有一个action函数指针，该指针指向了软中断处理函数。
    void    (*action)(struct softirq_action *);
};
```

**软中断向量表**

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```

由多个`softirq_action` 组成，目前linux共支持10类型的软中断，具体如下：

```C
enum
{
    HI_SOFTIRQ=0,//------------------------最高优先级的软中断类型，给高优先级的tasklet使用
    TIMER_SOFTIRQ,//-----------------------Timer定时器软中断
    NET_TX_SOFTIRQ,//----------------------发送网络数据包软中断
    NET_RX_SOFTIRQ,//----------------------接收网络数据包软中断
    BLOCK_SOFTIRQ,//-----------------------块设备软中断
    BLOCK_IOPOLL_SOFTIRQ,//----------------块设备软中断
    TASKLET_SOFTIRQ,//---------------------专门为tasklet机制准备的软中断
    SCHED_SOFTIRQ,//-----------------------进程调度以及负载均衡软中断
    HRTIMER_SOFTIRQ,//---------------------高精度定时器软中断
    RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq*/// --RCU服务软中断

    NR_SOFTIRQS
};
```

在`enum`中出现的顺序就是软中断的优先级，优先级越高越先被处理。

**软中断状态寄存器** 

`irq_cpustat_t  __softirq_pending`，其实就是一个`unsigned int`类型的变量，用于记录软中断是否发生，目前只用到了前10位。该变量并非全系统共用，而是每个CPU都有。

## 1.3 软中断的注册与激活

**软中断注册**

``` c
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
    softirq_vec[nr].action = action;
}
//例如: open_softirq(NET_TX_SOFTIRQ, net_tx_action);
```

`open_softirq`由内核提供，驱动程序可通过调用该函数将软中断服务程序软中断向量号绑定（bind）。

**软中断的激活**

```C
void raise_softirq(unsigned int nr)
{
	unsigned long flags;

	local_irq_save(flags);
	raise_softirq_irqoff(nr);//该函数完成软中断的激活
	local_irq_restore(flags);
}

inline void raise_softirq_irqoff(unsigned int nr)
{
	__raise_softirq_irqoff(nr);//将软中断状态寄存器 __softirq_pending的第nr（nr从0开始）位 置为1
	if (!in_interrupt())	   //判断当前调用是否在中断上下文中
		wakeup_softirqd();     //若不是在中断环境中则激活ksoftirqd线程处理软中断
}
```

软中断的激活表示软中断发生，调用路径为`raise_softirq--->raise_softirq_irqoff` ，`raise_softirq_irqoff` 函数主要完成将**软中断状态寄存器中的nr位置为1**表示向量号为nr的软中断已经发生，然后判断当前调用是否在中断环境中，若不在中断环境中则唤起ksoftirqd线程处理软中断。

**注：软中断激活并不意味着软中断被立即处理。因为软中断的处理是异步的，激活仅仅是表示软中断发生，至于何时处理软中断下文将对此进行说明。**

## 1.4 软中断的处理

**何时处理软中断？**

首先，处理软中断的函数为**`__do_softirq()`,**此函数会在两个地方被调用：

+ 中断上半部完成时，会调用`irq_exit()` ，然后`irq_exit-->invoke_softirq-->__do_softirq()`。但是此处调用`__do_softirq()`是有**“条件”**的,只有条件满足时才会处理软中断，具体条件下文将介绍。
+ 在ksoftirqd守护线程的执行函数中。

**irq_exit中处理软中断**

```c
void irq_exit(void)
{
		.......//省略与软中断处理无关的代码
	preempt_count_sub(HARDIRQ_OFFSET);//修改preempt_count标志，表示退出硬中断上下文环境
	if (!in_interrupt() && local_softirq_pending())//检查是否在中断上下文环境中
        										  //以及有无待处理的软中断
		invoke_softirq();//处理软中断
		.......//省略与软中断处理无关的代码
}
```

`irq_exit`中调用`preempt_count_sub(HARDIRQ_OFFSET)`为`hardirq_count`减去1,`hardirq_count`表示中断嵌套的层数，0表示没有中断发生,该变量通常用于检查当前环境是否为中断上下文环境。`preempt_count_sub(HARDIRQ_OFFSET)`可以理解为退出当前硬中断上下文环境。

`in_interrupt()`会检查`softirq_count与hardirq_count`的值判断当前是否在软/硬中断的上下文环境中。

`local_softirq_pending()`会检查当前有无需要待处理的软中断。

`irq_exit`函数首先标记退出当前硬中断上下文环境，然后在检查当前是否在中断上下文环境；若是，则说明当前为嵌套中断（嵌套了硬中断或者抢占了软中断处理线程），此时不应该继续处理软中断；若否，则继续判断当前有无待处理的软中断，若有则调用`invoke_softirq`处理软中断。

```c
static inline void invoke_softirq(void)
{
	if (!force_irqthreads) {//检查有无强制线程化
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
		/*
		 * We can safely execute softirq on the current stack if
		 * it is the irq stack, because it should be near empty
		 * at this stage.
		 */
		__do_softirq();
#else
		/*
		 * Otherwise, irq_exit() is called on the task stack that can
		 * be potentially deep already. So call softirq in its own stack
		 * to prevent from any overrun.
		 */
		do_softirq_own_stack();
#endif
	} else {
		wakeup_softirqd();
	}
}
```

`invoke_softirq`函数首先检查有无中断处理强制线程化，若有则唤起软中断处理守护线程ksoftirqd处理软中断。中断处理强制线程化是linux编译时的一个预定义，当启用时会强制所有中断都由中断线程处理，目前linux安装时并不会开启该功能。

当没有启用强制中断线程化时，软中断的处理也分成两种情况，主要是和软中断执行的时候使用的stack相关。如果arch支持单独的IRQ STACK，这时候，由于要退出中断，因此irq stack已经接近全空了，因此直接调用`__do_softirq()` 处理软中断就OK了，否则就调用`do_softirq_own_stack`函数在软中断自己的stack上执行。

**注：irq_exit中执行软中断，此时的执行环境仍然在中断上下文中。也就是说硬中断还没有执行iret，没有恢复到进程上下文中。`__do_softirq()`函数是处理触发的所有软中断，因此并非在硬中断结束时只处理在硬中断中触发软中断**

**ksoftirqd中处理软中断**

`run_ksoftirqd`为ksoftirqd守护线程执行函数。

```c
static void run_ksoftirqd(unsigned int cpu){
	local_irq_disable();//关中断
	if (local_softirq_pending()) {//检查有无需要处理的软中断
		/*
		 * We can safely run softirq on inline stack, as we are not deep
		 * in the task stack here.
		 */
		__do_softirq();//处理软中断
		local_irq_enable();//开中断
		cond_resched();//调度
		return;
	}
	local_irq_enable();}
```

ksoftirqd守护线程会关闭中断然后检查有无待处理的软中断，若有则调用`__do_softirq()`处理软中断，然后开中断，然后调用`cond_resched()`尽可能保证其他线程也能被正常调度到。

注：ksoftirqd是能被硬件中断所抢占的，`run_ksoftirqd`函数中一开始关中断是为了防止在检查有无需要处理的软中断时被中断，`__do_softirq()`在处理软中断时会开中断。

**__do_softirq()**

```c
void __do_softirq(void)
{
	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;//MAX_SOFTIRQ_TIME为2ms
    int max_restart = MAX_SOFTIRQ_RESTART;			//MAX_SOFTIRQ_RESTART为10次
		……

    pending = local_softirq_pending();//－－获取softirq pending的状态

    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);//标识下面的代码是正在处理softirq

    cpu = smp_processor_id();
restart:
    set_softirq_pending(0); //－－－－清除pending标志

    local_irq_enable(); //－－－－－－打开中断，softirq handler是开中断执行的

    h = softirq_vec; //－－－－－－－获取软中断描述符指针
	
    //寻找pending中第一个被设定为1的bit，寻找顺序与软中断优先级一致
    while ((softirq_bit = ffs(pending))) {
        unsigned int vec_nr;
        int prev_count;

        h += softirq_bit - 1; //－－－－－－指向pending的那个软中断描述符

        vec_nr = h - softirq_vec;//－－－－获取soft irq number

        h->action(h);//-----指向softirq handler；真正执行中断服务程序

        h++;
        pending >>= softirq_bit;
    }

    local_irq_disable(); //－－－关闭本地中断

    pending = local_softirq_pending();//－－－再次获取 软中断状态寄存器的值
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() &&
            --max_restart)//判断是否超时、是否需要调度、是否超过restart最大次数
            goto restart;//继续循环

        wakeup_softirqd();
    }

    __local_bh_enable(SOFTIRQ_OFFSET);//－－－－－－标识softirq处理完毕

}
```

`__do_softirq`处理系统中的所有软中断（而不是针对某一特定的软中断）。函数执行过程中会先关闭中断，读取软中断状态寄存器的所有状态信息以获取哪一个软中断已经发生，**然后开中断按照优先级依次执行已经被激活的软中断的处理函数**，完成本轮处理后判定有无继续执行的**“条件”**，所谓的条件指的是下面三个条件的交集:

* `__do_softirq`执行时间**未超过2ms**。
* `restart`次数**不超过10次**。`restart`次数表示一共处理了几轮软中断，当处理完软中断状态寄存器中所有的已激活的软中断时算作1轮。
* 内核中**没有线程需要调度**。

**注：`__do_softirq`在获取软中断状态寄存器时是关中断的，在执行软中断处理函数时是开中断的，因此软中断处理过程中可以被硬中断打断，但是不同类型的软中断在处理过程中是按照优先级串行处理的，而且软中断不可以打断软中断。**

## 1.5 总结

​		目前linux内核支持10种类型的软中断，内核中维护一个软中断向量表用于记录着每个软中断的处理函数，驱动程序可以将软中断处理函数注册给内核。软中断处理过程中本着**“何处触发，何处处理”**的原则，软中断在哪个CPU上被触发就需要在哪个CPU上处理，每个CPU都有一个**“软中断状态寄存器”**（本质上是一个32bit的变量）记录着当前软中断的触发情况。

​		中断处理的上半部分可以通过`rasie_softirq`函数触发软中断，但是**触发软中断并不意味着软中断会被立即处理**。软中断会在两种场景下被处理：

1. `irq_exit`函数中。此时软中断会在硬中断返回前，且在满足一定**“条件”**时会被处理，这个条件指的是**（硬中断返回时有未处理的软中断） 且 （硬中断不是嵌套中断且未打断软中断处理线程）且（没有执行强制中断处理线程化）**。满足上述条件时会在硬中断返回前调用`__do_softirq`,`__do_softirq`每次调用最多占用2ms，且最多处理10轮软中断，当有线程需要切换时会停止执行。
2. ksoftirqd软中断守护线程中。该守护线程会先关闭中断去检查有无软中断需要处理，然后调用`__do_softirq`去处理已经触发的软中断。执行完`__do_softirq`会进行线程调度，以防止其他线程发生“饥饿”。





# 二 、tasklet

## 2.1何为tasklet?

tasklet是中断下半部分处理的另一种机制，它是**基于软中断**实现的，使用tasklet实现的中断下半部分的实际处理仍然是由软中断守护进程ksoftirqd或者中断上半部分完成后处理。软中断中向量号为 `HI_SOFTIRQ、TASKLET_SOFTIRQ` 号对应的软中断处理程序`tasklet_hi_action、tasklet_action`专门用于处理需要执行的tasklet的两个软中断的，这个两个软中断处理程序是在内核启动时就注册好的，不允许驱动程序注册。tasklet分为两种，分别是有高优先级tasklet和低优先级tasklet，二者只不过在执行时高优先级的会被优先执行（因为高优先级的tasklet对应的软中断优先级高于低优先级tasklet对应的软中断），其原理无差别。

## 2.2为何要有tasklet？

首先分析下通过软中断处理中断下半段有哪些缺点：

* **软中断处理函数较难开发（需要考虑并行执行的情况）**。软中断的处理本着“何处触发，何处处理”的原则，即软中段在哪个CPU上触发就在哪个CPU上处理。软中断一般由硬中断触发，硬中断可以发生在不同CPU上，因此不同CPU可能会同时执行软中断处理函数，所以对驱动开发而言，软中断处理函数需要考虑到并行执行的情况，就难免避不开考虑互斥等操作。
* **软中断的数量是有限的。**上文提到目前Linux支持10类型的软中断，除了内核专用的软中断以外驱动程序只有四个软中断可用（分别给网络设备和块设备用），其他驱动程序需要中断下半段处理时就不够用了。

针对上述问题，tasklet有如下几个特性：

* **tasklet处理程序在所有CPU上是串行的。**同一时刻某一tasklet的处理程序只能在一个CPU上执行。
* **linux 对tasklet无数量限制。**

## 2.3内核中tasklet相关的数据结构

**tasklet描述符**

```c
struct tasklet_struct
{
    struct tasklet_struct *next;//------------------多个tasklet串成一个链表。
    unsigned long state;//--------------------------TASKLET_STATE_SCHED表示tasklet已经
   								 //被调度，正准备运行；TASKLET_STATE_RUN表示tasklet正在运行中。
    atomic_t count;//-------------------------------0表示tasklet处于激活状态；
    												//非0表示该tasklet被禁止，不允许执行。
    void (*func)(unsigned long);//------------------该tasklet处理程序
    unsigned long data;//---------------------------传递给tasklet处理函数的参数
};
```

tasklet描述符中state记录tasklet的状态，当tasklet已经被调度时第`TASKLET_STATE_SCHED`（0位）位置为1，当tasklet正在被执行时第`TASKLET_STATE_RUN`(1位)被置为1。`atomic_t count`用于表示tasklet是否处于可运行状态，0表示可运行，1表示不可运行。`func、data`变量分别是函数指针和函数参数。

**tasklet链表**

```c
struct tasklet_head {
    struct tasklet_struct *head;
    struct tasklet_struct **tail;
};
```

每个CPU维护者两个tasklet链表（高优先级、低优先级各一个），链表由多个tasklet描述符组成，记录着等待执行的tasklet。

## 2.4 tasklet的调度

**tasklet的调度类似于软中断中的触发（激活）操作。**tasklet调度通常发生在中断上半部分，调度后并不立即执行，而是将tasklet描述符挂到当前CPU tasklet链表中，触发软中断，然后等待在软中断处理函数（`tasklet_action`）中去执行tasklet处理函数。

**tasklet 调度函数**

```C
static inline void tasklet_schedule(struct tasklet_struct *t)
{
    //置TASKLET_STATE_SCHED位，如果原来未被置位，则调用__tasklet_schedule()
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))  // ----说明(1)
        __tasklet_schedule(t);
}

void __tasklet_schedule(struct tasklet_struct *t)
{
    unsigned long flags;

    local_irq_save(flags);
    t->next = NULL;
    
    *__this_cpu_read(tasklet_vec.tail) = t;//将t挂入到tasklet_vec链表中
    __this_cpu_write(tasklet_vec.tail, &(t->next));
    raise_softirq_irqoff(TASKLET_SOFTIRQ);//触发 TASKLET_SOFTIRQ 软中断。
    local_irq_restore(flags);
}
```

`__tasklet_schedule(t)`负责将`tasklet t`挂到当前CPU的tasklet链表中，并触发一次`TASKLET_SOFTIRQ`软中断。

**说明（1）：**此处测试并置`TASKLET_STATE_SCHED`位（第0位）。当第0位没有置位时会将该位置位，并调用

`__tasklet_schedule(t)`，当第0位已经置位时，`tasklet_schedule`不会执行操作。这个地方**保证了每个tasklet在被执行完前只能分配给一个CPU执行。**该处如此设置主要应对如下这种情况：

​		当硬件A触发CPU0硬中断时，在硬中断服务程序中（硬件A的驱动程序）会初始化一个tasklet描述符并调用`tasklet_schedule`,由于第一次调用`tasklet_schedule`，此时会将tasklet挂到CPU0的tasklet链表中等待被执行。在该tasklet被执行之前，如果硬件A再次触发硬中断，且硬中断被CPU1响应，此时硬中断服务程序仍然会调用`tasklet_schedule`，但是tasklet已经被分配给CPU0，所以此次`tasklet_schedule`不会有任何操作，硬件A驱动中的tasklet不会被分配到CPU1上。

## 2.4 tasklet 处理程序的执行

**负责执行tasklet处理程序的函数**

``` C
static void tasklet_action(struct softirq_action *a)
{
    struct tasklet_struct *list;

    local_irq_disable();
    //在关中断情况下读取tasklet_vec表头作为临时链表list
    list = __this_cpu_read(tasklet_vec.head);
    __this_cpu_write(tasklet_vec.head, NULL);//重新初始化tasklet_vec
    __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
    local_irq_enable();

    while (list) {//开中断情况下遍历tasklet_vec链表,所以tasklet是开中断的
        struct tasklet_struct *t = list;

        list = list->next;

        if (tasklet_trylock(t)) {//如果返回false，表示当前tasklet已经在其他CPU上运行
            			//这一轮将会跳过此tasklet。确保同一个tasklet只能在一个CPU上运行。
            			
            if (!atomic_read(&t->count)) {//表示当前tasklet处于激活状态
                if (!test_and_clear_bit(TASKLET_STATE_SCHED,
                            &t->state))//清TASKLET_STATE_SCHED位；
                    				//如果原来没有被置位，则返回0，触发BUG()。
                    BUG();
                t->func(t->data);//执行当前tasklet处理函数
                tasklet_unlock(t);
                continue;//跳到while继续遍历余下的tasklet
            }
            tasklet_unlock(t);
        }
        local_irq_disable();//此种情况说明即将要执行tasklet时，发现该tasklet已经在别的CPU上运行。
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;//把当前tasklet挂入到当前CPU的tasklet_vec中，												//等待下一次触发时再执行。
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        __raise_softirq_irqoff(TASKLET_SOFTIRQ);//再次置TASKLET_SOFTIRQ位
        local_irq_enable();
    }
}
```

`tasklet_action`是`TASKLET_SOFTIRQ`软中断的处理程序。`tasklet_action`中先关中断获取待处理的tasklet链表，并清空该链表，然后开中断。之后遍历tasklet链表依次执行链表中每个tasklet的处理函数。在执行每个tasklet的处理函数前会调用`tasklet_trylock(t)`,该函数主要是测试`tasklet->state`中的第`TASKLET_STATE_RUN`位然后置位。**当第`TASKLET_STATE_RUN`位已经被置位时，说明该tasklet已经在其他CPU上执行了(*问题1*)。**然后检查`tasklet->count`来得知该tasklet是否可执行，然后测试并清空`tasklet->state`中的第`TASKLET_STATE_SCHED`位,再然后执行该tasklet的处理函数**（`t->func(t->data)`此时是开中断的）**。执行完`t->func(t->data)`后会清掉第`TASKLET_STATE_RUN`位。

**问题1：**既然通过测试第`TASKLET_STATE_SCHED`位可以保证该tasklet不被再次分配到其他CPU上，为什么运行前再检测第`TASKLET_STATE_RUN`位来保证当前tasklet不被其他CPU运行呢？

**答：**测试并置位第`TASKLET_STATE_SCHED`位，仅仅能保证正常情况下tasklet不被分配到其他CPU上。因为在执行`t->func(t->data)`（tasklet处理函数）时第`TASKLET_STATE_SCHED`位已经被清掉，而且此时是开中断的，所以执行`t->func(t->data)`的过程（假设tasklet对应的硬件为HW0，此时CPU0正在执行其处理程序）中有可能会被其他中断中断掉，然后CPU1若来了HW0的硬中断，因此时的tasklet第`TASKLET_STATE_SCHED`位已经被清掉，所以中断服务程序可能会将该tasklet添加到CPU1的 tasklet链表中，这样就有可能发生多个CPU并行执行tasklet服务程序的情况。所以在执行`t->func(t->data)`时要检查并置位第`TASKLET_STATE_RUN`位。

## 2.5总结

本文以低优先级的tasklet为例，阐述了其具体原理。高优先级的tasklet与低优先级的tasklet原理上没有区别，只不过高优先级tasklet的处理在`tasklet_hi_action`函数中（`HI_SOFTIRQ`软中断的处理程序）。高优先级tasklet的描述符为`tasklet_hi_struct`,调度函数为`tasklet_hi_schedule`。

在处理中断下半部分时，tasklet与软中断机制的区别在于tasklet处理程序在所有CPU间是串行执行的，且系统中没有tasklet的数量限制。每个tasklet描述符除了有tasklet处理程序及其参数外，还有一个state变量和count变量，state记录当前tasklet有没有被调度和运行，通过检查state变量的第0位和第1位可以得知该tasklet有没有被调度和执行。若tasklet被调度但未被执行时，再次调用调度函数不会有任何操作，这保证了tasklet描述符在正常情况下不会被分到其他CPU tasklet链表中。由于执行tasklet处理程序时是开中断的，所以tasklet描述符还是有可能被分到多个CPU tasklet链表中的，所以在执行tasklet处理程序前会检查state变量的第1位，以判断有无被其他CPU执行。

