---
title: 20 芯片级移植需要的驱动
---

# 内核节拍驱动
当前Linux多采用无节拍方案，并支持高精度定时器，内核的配置一般会使能NO_HZ（即无节拍，或者说动态节拍）和HIGH_RES_TIMERS。无节拍并不是说系统中没有时钟节拍，而是说这个节拍不再像以前那样周期性地产生。

SoC底层的定时器被实现为一个clock_event_device和clocksource形式的驱动。在clock_event_device结构体中，实现其set_mode（）和set_next_event（）成员函数；在clocksource结构体中，主要实现read（）成员函数。而在定时器中断服务程序中调用clock_event_device的event_handler（）成员函数。
```c
static irqreturn_t xxx_timer_interrupt(int irq, void *dev_id)
{
	struct clock_event_device *ce = dev_id;
	// event_handler（）成员函数的具体工作是Linux内核根据Linux内核配置和运行情况自行设置的
	ce->event_handler(ce);
	
	return IRQ_HANDLED;
}

/* read 64-bit timer counter */
static cycle_t xxx_timer_read(struct clocksource *cs)
{
	u64 cycles;
	
	/* read the 64-bit timer counter */
	cycles = readl_relaxed(xxx_timer_base + LATCHED_HI);
	cycles=(cycles<<32)|readl_relaxed(xxx_timer_base + LATCHED_LO);
	
	return cycles;
}

static int xxx_timer_set_next_event(unsigned long delta,struct clock_event_device *ce)
{
	unsigned long now, next;
	now = readl_relaxed(xxx_timer_base + LATCHED_LO);
	next = now + delta;
	writel_relaxed(next, xxx_timer_base + SIRFSOC_TIMER_MATCH_0);
	...
}

static void xxx_timer_set_mode(enum clock_event_mode mode, struct clock_event_device *ce)
{
	switch (mode) {
	case CLOCK_EVT_MODE_PERIODIC:
	…
	case CLOCK_EVT_MODE_ONESHOT:
	…
	case CLOCK_EVT_MODE_SHUTDOWN:
	…
	case CLOCK_EVT_MODE_UNUSED:
	case CLOCK_EVT_MODE_RESUME:
	break;
	}
}
static struct clock_event_device xxx_clockevent = {
	.name = "xxx_clockevent"
	.rating = 200,
	.features = CLOCK_EVT_FEAT_ONESHOT,
	.set_mode = xxx_timer_set_mode,
	.set_next_event = xxx_timer_set_next_event,
};

static struct clocksource xxx_clocksource = {
	.name = "xxx_clocksource"
	.rating = 200,
	.mask = CLOCKSOURCE_MASK(64),
	.flags = CLOCK_SOURCE_IS_CONTINUOUS,
	.read = xxx_timer_read,
	.suspend = xxx_clocksource_suspend,
	.resume = xxx_clocksource_resume,
};

static struct irqaction xxx_timer_irq = {
	.name = "xxx_tick"
	.flags = IRQF_TIMER,
	.irq = 0,
	.handler = xxx_timer_interrupt,
	.dev_id = &xxx_clockevent,
};

static void __init xxx_clockevent_init(void)
{
	clockevents_calc_mult_shift(&xxx_clockevent, CLOCK_TICK_RATE, 60);
	// 设置xxx_timer_set_next_event参数delta能接受的值范围
	xxx_clockevent.max_delta_ns = clockevent_delta2ns(-2, &xxx_clockevent);
	xxx_clockevent.min_delta_ns = clockevent_delta2ns(2, &xxx_clockevent);
	
	xxx_clockevent.cpumask = cpumask_of(0);
	clockevents_register_device(&xxx_clockevent);
}

/* initialize the kernel jiffy timer source */
static void __init xxx_timer_init(void)
{
	…
	BUG_ON(clocksource_register_hz(&xxx_clocksource, CLOCK_TICK_RATE));
	BUG_ON(setup_irq(xxx_timer_irq.irq, &xxx_timer_irq));
	xxx_clockevent_init();
}
struct sys_timer xxx_timer = {
	.init = xxx_timer_init,
};
```
这些函数的结合使得ARM Linux内核底层所需要的时钟得以运行。下面举一个典型的场景，假定定时器的晶振时钟频率为1MHz（即计数器每加1等于1μs），应用程序通过nanosleep（）API睡眠100μs，内核会据此换算出下一次定时器中断的delta值为100，并间接调用xxx_timer_set_next_event（）去设置硬件让其在100μs后产生中断。100μs后，中断产生，xxx_timer_interrupt（）被调用，event_handler（）会间接唤醒睡眠的进程并导致nanosleep（）函数返回，从而让用户进程继续。

对于多核处理器来说，一般的做法是给每个核分配一个独立的定时器，各个核根据自身的运行情况动态地设置自己时钟中断发生的时刻。而比较低效率的方法则是只给CPU0提供定时器，由CPU0将定时器中断通过IPI（Inter Processor
Interrupt，处理器间中断）广播到其他核。