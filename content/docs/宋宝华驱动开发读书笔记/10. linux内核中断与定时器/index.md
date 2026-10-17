---
title: 10. linux内核中断与定时器
---

linux中的中断不可嵌套。为了在中断执行时间尽量短和中断处理需完成的工作量大之间找到一个平衡点，Linux将中断处理程序分解为两个半部：上半部（Top Half）和下半部（BottomHalf）。上半部在关中断状态下处理紧急的事情，下半部在开中断下处理非紧急事务，可以被其他中断打断，处理完其他中断的上半部后，继续回来执行之前没执行完的下半部。
![Pasted image 20251214144205](assets/Pasted%20image%2020251214144205.png)
![Pasted image 20251214144220](assets/Pasted%20image%2020251214144220.png)
```c
// 申请中断
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,const char *name, void *dev);
// devm_开头的API申请的资源是内核自动管理的，无需显式释放
int devm_request_irq(struct device *dev, unsigned int irq, irq_handler_t handler,unsigned long irqflags, const char *devname, void *dev_id);
// 释放中断
void free_irq(unsigned int irq,void *dev_id);
// 中断源屏蔽
void disable_irq(int irq);  // 等待指定中断处理完毕返回
void disable_irq_nosync(int irq);  // 立即返回，不会等待指定中断处理完毕
void enable_irq(int irq);
// 屏蔽和恢复本CPU核的所有中断,local_开头的方法作用范围为本CPU核
#define local_irq_save(flags) ...
void local_irq_disable(void);
#define local_irq_restore(flags) ...
void local_irq_enable(void);
```

# 下半部（BottomHalf）
下半部有多种实现机制，Linux 中断下半部发展时间线：
- Linux 2.2 (1999)     → BH（Bottom Halves）- 2.2 版本的下半部，已完全移除
- Linux 2.3 (2000)     → Tasklets - 不推荐新代码使用，但有大量遗留驱动使用
- Linux 2.5 (2002)     → Softirqs - 内核核心基础设施，仍在活跃使用，驱动一般不直接用
- Linux 2.6.17 (2006)  → Threaded IRQs - 可线程化的中断处理，中断处理的现代方案
- Linux 3.14 (2014)    → Workqueues 重构 - 当前最推荐的通用延迟执行机制
- 至今                → 多种机制并存，各有适用场景

| 机制                | 执行上下文  | 可否睡眠            | 并发性              | 推荐程度     |
| ----------------- | ------ | --------------- | ---------------- | -------- |
| **Softirqs**      | 软中断上下文 | 不可睡眠            | 同类型串行            | 内核核心使用   |
| **Tasklets**      | 软中断上下文 | 不可睡眠            | 同类型串行            | 不推荐新代码   |
| **Workqueues**    | 进程上下文  | 可以睡眠，释放时使用完成量同步 | 串行，可手动创建内核线程独立运行 | 推荐用于简单任务 |
| **Threaded IRQs** | 进程上下文  | 可以睡眠，释放时使用完成量同步 | 每个IRQ运行一个独立线程    | 推荐用于耗时任务 |

## softirq
驱动不宜直接使用softirq
```c
void open_softirq(int nr, void (*action)(struct softirq_action));  // 注册软中断
raise_softirq(unsigned int nr);  // 触发软中断
local_bh_enable()  // 禁止和使能软中断及tasklet
local_bh_disable()
```
## tasklet
```c
/* 定义tasklet和下半部函数并将它们关联 */
void xxx_do_tasklet(unsigned long);
DECLARE_TASKLET(xxx_tasklet, xxx_do_tasklet, 0);
/* 中断处理下半部 */
void xxx_do_tasklet(unsigned long)
{
	...
}
/* 中断处理上半部 */
irqreturn_t xxx_interrupt(int irq, void *dev_id)
{
	...
	tasklet_schedule(&xxx_tasklet);
	...
}
/* 设备驱动模块加载函数 */
int __init xxx_init(void)
{
	...
	/* 申请中断 */
	result = request_irq(xxx_irq, xxx_interrupt, IRQF_xxx, "xxx, NULL);
	...
	return IRQ_HANDLED;
}
/* 设备驱动模块卸载函数 */
void __exit xxx_exit(void)
{
	...
	/* 释放中断 */
	free_irq(xxx_irq, xxx_interrupt);
	tasklet_kill(xxx_tasklet);
	...
}
```

## Workqueues
一个workqueues中有多个 work，前一个 work 没处理完会影响后面的 work。解决方法有很多种，比如干脆自己创建一个内核线程，不跟别的 work 凑在一块了。
![Pasted image 20251213183156](assets/Pasted%20image%2020251213183156.png)
![Pasted image 20251213183212](assets/Pasted%20image%2020251213183212.png)
```c
/* 定义工作队列和关联函数 */
struct work_struct xxx_wq;
/* 中断处理下半部 */
void xxx_do_work(struct work_struct *work)
{
	...
}
/*中断处理上半部*/
irqreturn_t xxx_interrupt(int irq, void *dev_id)
{
	...
	schedule_work(&xxx_wq);
	...
	return IRQ_HANDLED;
}
/* 设备驱动模块加载函数 */
int xxx_init(void)
{
	...
	/* 申请中断 */
	result = request_irq(xxx_irq, xxx_interrupt, IRQF_xxx, "xxx", NULL);
	...
	/* 初始化工作队列 */
	INIT_WORK(&xxx_wq, xxx_do_work);
	...
}
/* 设备驱动模块卸载函数 */
void xxx_exit(void)
{
	...
	/* 释放中断 */
	free_irq(xxx_irq, xxx_interrupt);
	cancel_work_sync(&xxx_wq);
	...
}
```

## Threaded IRQs
threaded irq，线程化的中断处理。上半部用来处理紧急的事情，下半部用一个内核线程来处理，这个内核线程专用于这个中断。
```c
int request_threaded_irq(unsigned int irq, 
						irq_handler_t handler,
						irq_handler_t thread_fn,
						unsigned long flags, 
						const char *name, 
						void *dev);
int devm_request_threaded_irq(struct device *dev, 
							unsigned int irq,
							irq_handler_t handler, 
							irq_handler_t thread_fn,
							unsigned long irqflags,
							const char *devname,
							void *dev_id);
```
![Pasted image 20251213183657](assets/Pasted%20image%2020251213183657.png)
```c
// 下半部
static irqreturn_t my_threaded_handler(int irq, void *dev_id)
{
    // 在进程上下文中运行，可以睡眠
    process_data(dev_id);
    return IRQ_HANDLED;
}
// 上半部中断函数
static irqreturn_t my_hard_handler(int irq, void *dev_id)
{
    // 快速处理，在中断上下文中运行
    if (!data_ready(dev_id))
        return IRQ_NONE;
        
    return IRQ_WAKE_THREAD;
}

// 注册和释放线程化中断
request_threaded_irq(irq, my_hard_handler, my_threaded_handler,
                     IRQF_ONESHOT, "my_device", dev);
free_irq(irq, dev);
// 可以调用下列函数等待my_threaded_handler执行完毕
synchronize_irq(unsigned int irq);
```

# linux中断控制器驱动
![Pasted image 20251214160020](assets/Pasted%20image%2020251214160020.png)
# 内核定时器
软件意义上的定时器最终依赖硬件定时器来实现，内核在时钟中断发生后检测各定时器是否到期，到期后的定时器处理函数将作为软中断在下半部执行。实质上，时钟中断处理程序会唤起TIMER_SOFTIRQ软中断，运行当前处理器上到期的所有定时器。
```c
/* xxx设备结构体 */
struct xxx_dev {
	struct cdev cdev;
	...
	struct timer_list xxx_timer; /* 设备要使用的定时器 */
};

/* xxx驱动中的中断处理函数 */
xxx_interrupt(…)
{
	struct xxx_dev *dev = filp->private_data;
	...
	/* 设置定时器 */
	// setup_timer(&dev->xxx_timer, xxx_do_timer, dev); // linux_version < 4.14
	timer_setup(&dev->xxx_timer, xxx_do_timer, 0);
	/* 修改定时时间 */
	mod_timer(&dev->xxx_timer, jiffies + msecs_to_jiffies(delay));
	...
}
/* 定时器处理函数 */
static void xxx_do_timer(struct timer_list * timer)
{
	struct xxx_dev *dev = from_timer(sturct xxx_dev, timer, xxx_timer);
	/* 删除定时器 */
	del_timer (&dev->xxx_timer);
	// 处理事件
	...
}
```

## hrtimer高精度定时器
Linux内核支持tickless和NO_HZ模式后，也包含对hrtimer（高精度定时器）的支持，它可以支持到微秒级别的精度。
```c
static enum hrtimer_restart snd_hrtimer_callback(struct hrtimer *hrt)
{
	...
	// 将时间增加poll_time_ns纳秒后再次触发
	hrtimer_forward_now(hrt, ns_to_ktime(iprtd->poll_time_ns));
	return HRTIMER_RESTART;
}
static int snd_imx_open(struct snd_pcm_substream *substream)
{
	...
	// 初始化高精度定时器
	hrtimer_init(&iprtd->hrt, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
	iprtd->hrt.function = snd_hrtimer_callback;
	hrtimer_start(&iprtd->hrt, ns_to_ktime(iprtd->poll_time_ns), HRTIMER_MODE_REL);
	...
	return 0;
}
static int snd_imx_close(struct snd_pcm_substream *substream)
{
	...
	// 清除注册的定时器
	hrtimer_cancel(&iprtd->hrt);
	...
}
```

## delayed_work
对于周期性的任务，除了定时器以外，在Linux内核中还可以利用一套封装得很好的快捷机制，其本质是利用工作队列和定时器实现，这套快捷机制就是delayed_work。
```c
1struct delayed_work {
2 struct work_struct work;
3 struct timer_list timer;
4
5 /* target workqueue and CPU ->timer uses to queue ->work */
6 struct workqueue_struct *wq;
7 int cpu;
8};
// 在delay延迟后执行dealyed_work中的函数
int schedule_delayed_work(struct delayed_work *work, unsigned long delay);
// 取消delayed_work
int cancel_delayed_work(struct delayed_work *work);
int cancel_delayed_work_sync(struct delayed_work *work);
```

# 内核延时
## 忙等待延时
它根据CPU频率进行一定次数的循环，通常用于配合硬件上的短延时需求。
```c
// 短延时
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);
// 长延时
#define time_after(a,b) \
(typecheck(unsigned long, a) && \
typecheck(unsigned long, b) && \
((long)(b) - (long)(a) < 0))
#define time_before(a,b) time_after(b,a)
while(time_before(jiffies, delay));
```
## 睡眠延迟
睡眠延迟是在等待的时间到来之前进程处于睡眠状态，CPU资源被其他进程使用。schedule_timeout（）可以使当前任务休眠至指定的jiffies之后再重新被调度执行，schedule_timeout（）的实现原理是向系统添加一个定时器，在定时器处理函数中唤醒与参数对应的进程。
```c
schedule_timeout(timeout);
schedule_timeout_uninterruptible(timeout);
schedule_timeout_interruptible(timeout);
void msleep(unsigned int millisecs);
unsigned long msleep_interruptible(unsigned int millisecs);
void ssleep(unsigned int seconds);
// 使用等待队列等待条件成立或超时
long wait_event_timeout(wait_queue_head_t queue, 
                       condition, 
                       long timeout);
```
msleep和msleep_interruptible在本质上都是依靠包含了schedule_timeout的schedule_timeout_uninterruptible和schedule_timeout_interruptible来实现的。
