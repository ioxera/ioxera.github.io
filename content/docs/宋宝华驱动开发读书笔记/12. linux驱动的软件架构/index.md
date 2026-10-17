---
title: 12. linux驱动的软件架构
---

在Linux设备驱动模型中，需关心总线、设备和驱动这3个实体，总线将设备和驱动绑定。class类为底层驱动在用户空间提供了跨所有设备类型的统一抽象，按功能而非技术实现分组。
```
虚拟文件系统（VFS）(/dev/xxx, /sys/class/, socket接口)
		↑ 文件系统接口file_operations/SOCKET/class类
字符设备/块设备/网络设备 (内核技术分类，作为应用层和底层驱动的接口)
		↑ 注册并实现接口
input/rtc/leds/keys/gpio/pwm/framebuffer/tty/misc 等功能子系统层(底层设备被抽象成不同类别，供上层使用，可选)
		↑ 注册并实现接口
spi/i2c/usb 设备驱动（客户端驱动，具体设备的驱动,可选）
	    ↑ API
spi/i2c/usb 核心层（作为客户端驱动和主机驱动连接的桥梁，提供数据结构和编程框架，可选）
		↑ 注册并实现接口
spi/i2c/usb 总线控制器驱动（主机驱动，获取接口并设置寄存器，使用platform虚拟总线框架实现）
		↑ 接口具体连接定义
   设备定义/DTS
```
# platform设备驱动框架
一个现实的Linux设备和驱动通常都需要挂接在一种总线上，对于本身依附于PCI、USB、I2C、SPI等的设备而言，这自然不是问题，但是在嵌入式系统里面，在SoC内部集成的I 2C、RTC、LCD、看门狗等控制器、挂接在SoC内存空间的外设等却不依附于此类总线。基于这一背景，Linux发明了一种虚拟的总线，称为platform总线，相应的设备称为platform_device，而驱动称为platform_driver。
```c
struct platform_driver {
	int (*probe)(struct platform_device *);
	int (*remove)(struct platform_device *);
	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
	bool prevent_deferred_probe;
};
```
直接填充platform_driver的suspend（）、resume（）做电源管理回调的方法目前已经过时，较好的做法是实现platform_driver的device_driver中的dev_pm_ops结构体成员。device_driver描述了各种xxx_driver（xxx是总线名）在驱动意义上的一些共性。
```c
struct bus_type platform_bus_type = {
	.name = "platform",
	.dev_groups = platform_dev_groups,
	.match = platform_match,
	.uevent = platform_uevent,
	.pm = &platform_dev_pm_ops,
};
```
bus_type的实例platform_bus_type将platform_device和platform_driver绑定到一起，在platform_device和platform_driver注册的时候，platform_match函数被调用以匹配platform_device和platform_driver。
```c
struct platform_device {
	const char *name;
	int id;
	boo id_auto;
	struct device dev;
	u32 num_resources;
	struct resource *resource;
	
	const struct platform_device_id *id_entry;
	char *driver_override; /* Driver name to force a match */
	
	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;
	
	/* arch specific additions */
	struct pdev_archdata archdata;
};
```
对于Linux 2.6ARM平台而言，对platform_device的定义通常在BSP的板文件中实现，在板文件中，将platform_device归纳为一个数组，最终通过platform_add_devices（）函数统一注册。Linux 3.x之后，而倾向于根据设备树中的内容自动解析为platform_device。platform_device.dev.platform_data成员可用于保存一些附加的开发板特定的配置信息，其结构由具体的驱动定义，驱动通过dev_get_platdata函数获取此信息。
platform_device的resource成员描述了开发板上的资源。在具体的设备驱动中通过platform_get_resource（）这样的API来获取。
```c
struct resource {
	resource__size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	struct resource *parent, *sibling, *child;
};
```
![Pasted image 20260104195606](assets/Pasted%20image%2020260104195606.png)
# 设备驱动的子系统分层架构
在分层设计的时候，Linux内核大量使用了面向对象的设计思想。例如platform_driver中包含device_driver实例来抽象设备驱动的共性，platform_driver和device_driver结构中都包含probe/remove/shutdown等回调，可在device_driver的回调中实现通用功能，如有特殊需求，platform_driver的同名回调中可实现设备特定的功能来覆盖device_driver中的回调。platform_device中的device类型实例也是同样的功能。
在设备驱动方面，往往为同类的设备设计了一个框架，而框架中的核心层则实现了该设备通用的一些功能。同样的，如
果具体的设备不想使用核心层的函数，也可以重写。
```c
return_type core_funca(xxx_device * bottom_dev, param1_type param1, param1_type param2)
{
	if (bottom_dev->funca)
		return bottom_dev->funca(param1, param2);
	/* 核心层通用的funca代码 */
	...
	/*通用的步骤代码A */
	typea_dev_commonA();
	...
	/* 底层操作ops1 */
	bottom_dev->funca_ops1();
	/*通用的步骤代码B */
	typea_dev_commonB();
	...
	/* 底层操作ops2 */
	bottom_dev->funca_ops2();
	/*通用的步骤代码C */
	typea_dev_commonB();
	...
	/** 底层操作ops3*/
	bottom_dev->funca_ops3();
}
```
这样的分层设计在Linux的input、RTC、MTD、I 2C、SPI、tty、USB等诸多类型设备驱动中广泛使用。
## input设备驱动
输入设备（如按键、键盘、触摸屏、鼠标等）是典型的字符设备，其一般的工作机理是底层在按键、触摸等动作发送时产生一个中断（或驱动通过Timer定时查询），然后CPU通过SPI、I2C或外部存储器总线读取键值、坐标等数据，并将它们包装成input_event结构并放入一个缓冲区，字符设备驱动管理该缓冲区，而驱动的read（）接口让用户可以读取键值、坐标等数据。在这些工作中，只是中断、读键值/坐标值是与设备相关的，而输入事件的缓冲区管理以及字符设备驱动的file_operations接口则对输入设备是通用的。基于此，内核设计了输入子系统，由核心层处理公共的工作。
![Pasted image 20251118190259](assets/Pasted%20image%2020251118190259.png)
```c
struct input_dev *input_allocate_device(void);
void input_free_device(struct input_dev *dev);
int __must_check input_register_device(struct input_dev *);
void input_unregister_device(struct input_dev *);
// 事件上报相关函数
struct input_event {
	struct timeval time;
	__u16type;
	__u16code;
	__s32value;
};
/* 报告指定type、code的输入事件 */
void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value);
/* 报告键值 */
void input_report_key(struct input_dev *dev, unsigned int code, int value);
/* 报告相对坐标 */
void input_report_rel(struct input_dev *dev, unsigned int code, int value);
/* 报告绝对坐标 */
void input_report_abs(struct input_dev *dev, unsigned int code, int value);
/* 报告同步事件 */
void input_sync(struct input_dev *dev);
```
drivers/input/keyboard/gpio_keys.c基于input架构实现了一个通用的GPIO按键驱动。该驱动是基于platform_driver架构的，名为“gpio-keys”。可以将gpio_keys看做gpio key的虚拟板载控制器驱动。它将与硬件相关的信息（如使用的GPIO号，按下和抬起时的电平等）屏蔽在板文件platform_device的platform_data中，因此该驱动可应用于各个处理器，具有良好的跨平台性。在具体的gpio按键驱动注册输入设备后，按键驱动的核心工作只剩下在按键、触摸等人为动作发生时报告事件。
```c
static irqreturn_t gpio_keys_irq_isr(int irq, void *dev_id)
{
	struct gpio_button_data *bdata = dev_id;
	const struct gpio_keys_button *button = bdata->button;
	struct input_dev *input = bdata->input;
	unsigned long flags;
	
	BUG_ON(irq != bdata->irq);
	spin_lock_irqsave(&bdata->lock, flags);
	 if (!bdata->key_pressed) {
		 if (bdata->button->wakeup)
			 pm_wakeup_event(bdata->input->dev.parent, 0);
		
		 input_event(input, EV_KEY, button->code, 1);
		 input_sync(input);
		
		 if (!bdata->timer_debounce) {
		 input_event(input, EV_KEY, button->code, 0);
		 input_sync(input);
		 goto out;
	 }
	 bdata->key_pressed = true;
	 }
	 if (bdata->timer_debounce)
	 mod_timer(&bdata->timer,
	 jiffies + msecs_to_jiffies(bdata->timer_debounce));
out:
	 spin_unlock_irqrestore(&bdata->lock, flags);
	 return IRQ_HANDLED;
}
```
## RTC设备驱动
RTC设备核心层的架构和input层基本一致。RTC是一种典型字符设备，作为一种字符设备驱动，RTC需要有file_operations中接口函数的实现，如open（）、release（）、read（）、poll（）、ioctl（）等，而典型的IOCTL包括RTC_SET_TIME、RTC_ALM_READ、RTC_ALM_SET、RTC_IRQP_SET、RTC_IRQP_READ等，这些对于所有的RTC是通用的，只有底层的具体实现是与设备相关的。drivers/rtc/rtc-dev.c实现了RTC驱动通用的字符设备驱动层，它实现了file_opearations的成员函数以及一些通用的关于RTC的控制代码，并向底层导出rtc_device_register（）、rtc_device_unregister（）以注册和注销RTC；导出rtc_class_ops结构体以描述底层的RTC硬件操作。底层的RTC驱动不再需要关心RTC作为字符设备驱动的具体实现，也无需关心一些通用的RTC控制逻辑。
![Pasted image 20251118192352](assets/Pasted%20image%2020251118192352.png)
```c
static const struct rtc_class_ops s3c_rtcops = {
	.read_time = s3c_rtc_gettime,
	.set_time = s3c_rtc_settime,
	.read_alarm = s3c_rtc_getalarm,
	.set_alarm = s3c_rtc_setalarm,
	.proc = s3c_rtc_proc,
	.alarm_irq_enable = s3c_rtc_setaie,
};
static int s3c_rtc_probe(struct platform_device *pdev)
{
	...
	rtc = devm_rtc_device_register(&pdev->dev,"s3c", &s3c_rtcops,THIS_MODULE);
	...
}
```

## framebuffer设备驱动
为Linux帧缓冲设备驱动的主要结构，帧缓冲设备提供给用户空间的file_operations结构体由drivers/video/fbdev/core/fbmem.c中的file_operations提供，而特定帧缓冲设备fb_info结构体的注册、注销以及其中成员的维护，尤其是fb_ops中成员函数的实现则由对应的xxxfb.c文件实现，fb_ops中的成员函数最终会操作LCD控制其硬件寄存器。
![Pasted image 20251118192844](assets/Pasted%20image%2020251118192844.png)

## 终端设备驱动
终端是一种字符型设备，它有多种类型，通常使用tty（Teletype）来简称各种类型的终端设备。对于嵌入式系统而言，最普遍采用的是UART（Universal Asynchronous Receiver/Transmitter）串行端口，日常生活中简称串口。
tty的层次结构包含tty核心tty_io.c、tty线路规程n_tty.c（实现N_TTY线路规程）和tty驱动实例xxx_tty.c，tty线路规程的工作是以特殊的方式格式化从一个用户或者硬件收到的数据，这种格式化常常采用一个协议转换的形式。tty_io.c本身是一个标准的字符设备驱动，它对上有字符设备的职责，实现file_operations成员函数。但是tty核心层对下又定义了tty_driver的架构，这样tty设备驱动的主体工作就变成了填充tty_driver结构体中的成员，实现其中的tty_operations的成员函数，而不再是去实现file_operations这一级的工作。
![Pasted image 20251118193134](assets/Pasted%20image%2020251118193134.png)
![Pasted image 20251118193308](assets/Pasted%20image%2020251118193308.png)
尽管一个特定的底层UART设备驱动完全可以遵循上述tty_driver的方法来设计，即定义tty_driver并实现tty_operations中的成员函数，但是鉴于串口之间的共性，Linux考虑在文件drivers/tty/serial/serial_core.c中实现了UART设备的通用tty驱动层（我们可以称其为串口核心层）。这样，UART驱动的主要任务就进一步演变成了实现serial-core.c中定义的一组uart_xxx接口而不是tty_xxx接口。串口核心层定义了新的uart_driver结构体和其操作集uart_ops。一个底层的UART驱动需要创建和通过uart_register_driver（）注册一个uart_driver而不是tty_driver。
![Pasted image 20251118193448](assets/Pasted%20image%2020251118193448.png)

## misc设备驱动
由于Linux驱动倾向于分层设计，所以各个具体的设备都可以找到它归属的类型，从而套到它相应的架构里面去，并且只需要实现最底层的那一部分。但是，也有部分字符设备，确实不知道它属于什么类型，一般推荐采用miscdevice框架结构。miscdevice本质上也是字符设备，具体miscdevice实例调用misc_register（）的时候又自动完成了device_create（）、获取动态次设备号的动作。miscdevice的主设备号是固定的，MISC_MAJOR定义为10。
```c
struct miscdevice {
	int minor;
	const char *name;
	const struct file_operations *fops;
	struct list_head list;
	struct device *parent;
	struct device *this_device;
	const char *nodename;
	umode_t mode;
};
int misc_register(struct miscdevice * misc);
int misc_deregister(struct miscdevice *misc);
```
miscdevice结构体内file_operations中的成员函数实际上是由drivers/char/misc.c中misc驱动核心层的misc_fops成员函数间接调用的，比如misc_open（）就会间接调用底层注册的miscdevice的fops->open。
# 总线控制器驱动和设备驱动的分离架构
Linux中的SPI、I2C、USB等子系统都利用了典型的把总线控制器驱动和外设驱动分离的想法，让总线控制器驱动只负责产生总线上的传输波形，核心层向上提供API接口，而外设驱动只是调用核心层的API让总线控制器驱动以适当的波形访问外设。外设的板级逻辑由设备定义或设备树提供给总线控制器。这样就实现了设备驱动和总线控制的解耦。
## SPI总线控制器驱动（主机驱动）
linux中使用spi_master结构体来描述一个SPI主机控制器驱动。分配、注册和注销SPI控制器驱动的API由SPI核心提供：
```c
struct spi_master * spi_alloc_master(struct device *host, unsigned size);
int spi_register_master(struct spi_master *master);
void spi_unregister_master(struct spi_master *master);
```
## SPI设备驱动（客户端驱动）
linux中使用spi_driver结构体来描述一个SPI外设驱动，spi_driver结构体和platform_driver结构体有极大的相似性，这几乎是一切客户端驱动的常用模板。SPI只是一种总线，spi_driver的作用只是将SPI外设挂接在该总线上，因此在spi_driver的probe（）成员函数中，将注册SPI外设本身所属设备驱动的类型。
```c
struct spi_driver {
	const struct spi_device_id *id_table;
	int (*probe)(struct spi_device *spi);
	int (*remove)(struct spi_device *spi);
	void (*shutdown)(struct spi_device *spi);
	int (*suspend)(struct spi_device *spi, pm_message_t mesg);
	int (*resume)(struct spi_device *spi);
	struct device_driver driver;
};
```
在SPI外设驱动中，当通过SPI总线进行数据传输的时候，使用了一套与CPU无关的统一的接口。一次完整的SPI传输流程可能包含多个spi_transfer，这些spi_transfer最终通过spi_message组织在一起。下面是非常典型的初始化spi_transferspi_message并进行SPI数据传输的例子，同时spi_write（）、spi_read（）也是SPI核心层的两个通用快捷API，在SPI外设驱动中可以直接调用它们进行简单的纯写和纯读操作。
```c
static inline int
spi_write(struct spi_device *spi, const u8*buf, size_t len)
{
	struct spi_transfer t = {
		.tx_buf = buf,
		.len = len,
	};
	struct spi_message m;
	
	spi_message_init(&m);
	spi_message_add_tail(&t, &m);
	return spi_sync(spi, &m);
}

static inline int
spi_read(struct spi_device *spi, u8*buf, size_t len)
{
	struct spi_transfer t = {
		.rx_buf = buf,
		.len = len,
	};
	struct spi_message m;
	
	spi_message_init(&m);
	spi_message_add_tail(&t, &m);
	return spi_sync(spi, &m);
}
```
