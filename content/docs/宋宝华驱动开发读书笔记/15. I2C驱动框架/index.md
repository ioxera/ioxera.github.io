---
title: 15. I2C驱动框架
---

I2C驱动由3部分组成，即I2C核心、I2C总线驱动和I2C设备驱动。
- I2C核心提供了I2C总线驱动和设备驱动的注册、注销方法，I2C通信方法（即Algorithm）上层与具体适配器无关的代码以及探测设备、检测设备地址的上层代码等。
- I2C总线驱动（也称主机驱动）主要包含I2C适配器数据结构i2c_adapter、I2C适配器的Algorithm数据结构i2c_algorithm和控制I2C适配器产生通信信号的函数。
- I2C设备驱动（也称为客户驱动）是对I2C硬件体系结构中设备端的实现，设备一般挂接在受CPU控制的I2C总线控制器上，通过I2C总线控制器与CPU交换数据。I2C设备驱动主要包含数据结构i2c_driver和i2c_client，
![Pasted image 20251128155718](assets/Pasted%20image%2020251128155718.png)

## I2C核心
i2c-core.c这个文件实现了I2C核心的功能以及`/proc/bus/i2c-*`接口。
## I2C总线驱动
- 提供I2C适配器的硬件驱动，探测、初始化I2C适配器（如申请I2C的I/O地址和中断号）、驱动CPU控
制的I2C适配器从硬件上产生各种信号以及处理I2C中断等。
- 提供I2C适配器的Algorithm，用具体适配器的xxx_xfer（）函数填充i2c_algorithm的master_xfer指针，并把i2c_algorithm指针赋值给i2c_adapter的algo指针。

```c
struct i2c_adapter {
	struct module *owner;
	unsigned int class; /* classes to allow probing for */
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;
	/* data fields that are valid for all devices */
	struct rt_mutex bus_lock;
	int timeout; /* in jiffies */
	int retries;
	struct device dev; /* the adapter device */
	int nr;
	char name[48];
	struct completion dev_released;
	struct mutex userspace_clients_lock;
	struct list_head userspace_clients;
	struct i2c_bus_recovery_info *bus_recovery_info;
};
1struct i2c_algorithm {
2 /* If an adapter algorithm can't do I2C-level access, set master_xfer
3 to NULL. If an adapter algorithm can do SMBus access, set
4 smbus_xfer. If set to NULL, the SMBus protocol is simulated
5 using common I2C messages */
6 /* master_xfer should return the number of messages successfully
7 processed, or a negative value on error */
8 int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
9 int num);
10 int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr,
11 unsigned short flags, char read_write,
12 u8 command, int size, union i2c_smbus_data *data);
13
14 /* To determine what the adapter supports */
15 u32 (*functionality) (struct i2c_adapter *);
16};

int i2c_add_adapter(struct i2c_adapter *adap);
void i2c_del_adapter(struct i2c_adapter *adap);

```
i2c_adapter对应于物理上的一个控制器，而i2c_algorithm对应一套通信方法。i2c_algorithm中的关键函数master_xfer用于产生I2C访问周期需要的信号，以i2c_msg（即I2C消息）为单位，其中的成员表明了I2C的传输地址、方向、缓冲区、缓冲区长度等信息。
```c
/*初始化I2C适配器所使用的硬件资源，如申请I/O地址、中断号、时钟等。
通过i2c_add_adapter（）添加i2c_adapter的数据结构*/
static int xxx_i2c_probe(struct platform_device *pdev)
{
	struct i2c_adapter *adap;
	...
	xxx_adpater_hw_init()
	adap->dev.parent = &pdev->dev;
	adap->dev.of_node = pdev->dev.of_node;
	adap->algo = &xxxx_algo;
	platform_set_drvdata(pdev，i2c_dev);
	i2c_set_adapdata(adap，i2c_dev);
	ret = i2c_add_adapter(adap);
	...
}

static int xxx_i2c_remove(struct platform_device *pdev)
{
	...
	xxx_adpater_hw_free()
	i2c_del_adapter(&dev->adapter);
	
	return 0;
}

static int i2c_adapter_xxx_xfer(struct i2c_adapter *adap, struct i2c_msg *msgs,int num)
{
	...
	for (i = 0; i < num; i++) {
	i2c_adapter_xxx_start(); /* 产生开始位 */
	/*读消息 */
	if (msgs[i]->flags &I2C_M_RD) {
		i2c_adapter_xxx_setaddr((msg->addr << 1) | 1); /* 发送从设备读地址 */
		i2c_adapter_xxx_wait_ack(); /* 获得从设备的ack */
		i2c_adapter_xxx_readbytes(msgs[i]->buf, msgs[i]->len); /* 读取msgs[i] ->len
		长的数据到msgs[i]->buf */
	} else { /* 写消息 */
		i2c_adapter_xxx_setaddr(msg->addr << 1); /* 发送从设备写地址 */
		i2c_adapter_xxx_wait_ack(); /* 获得从设备的ack */
		i2c_adapter_xxx_writebytes(msgs[i]->buf, msgs[i]->len); /* 读取msgs[i]->len
		长的数据到msgs[i]->buf */
	}
	}
	i2c_adapter_xxx_stop(); /* 产生停止位 */
}
// 返回algorithm所支持的通信协议,如I2C_FUNC_I2C、I2C_FUNC_10BIT_ADDR、I2C_FUNC_SMBUS_READ_BYTE、I2C_FUNC_SMBUS_WRITE_BYTE
u32 xxxx_functionality (struct i2c_adapter *){
	return I2C_FUNC_I2C;
}

static const struct i2c_algorithm xxxx_algo = {
	.master_xfer = xxxx_xfer,
	.smbus_xfer = NULL,
	.functionality = xxxx_functionality,
}

static const struct of_device_id xxx_i2c_of_match[] = {
	{.compatible = "vendor,xxx-i2c",},
	{},
};
MODULE_DEVICE_TABLE(of, xxx_i2c_of_match);

static struct platform_driver xxx_i2c_driver = {
	.driver = {
		.name = "xxx-i2c",
		.owner = THIS_MODULE,
		.of_match_table = xxx_i2c_of_match,
	},
	.probe = xxx_i2c_probe,
	.remove = xxx_i2c_remove,
};
module_platform_driver(xxx_i2c_driver);
```
## I2C设备驱动
- 实现I2C设备驱动中的i2c_driver接口，用具体的yyy_probe（）、yyy_remove（）、yyy_suspend（）、yyy_resume（）函数指针和i2c_device_id设备ID表赋值给i2c_driver的probe、remove、suspend、resume和id_table指针。
- 实现I2C设备所对应类型的具体驱动，i2c_driver只是实现设备与总线的挂接，而挂接在总线上的设备则千差万别。例如，如果是字符设备，就实现文件操作接口，即实现具体设备yyy的yyy_read（）、yyy_write（）和yyy_ioctl（）函数等；如果是声卡，就实现ALSA驱动。
```c
struct i2c_driver {
	unsigned int class;
	/* Notifies the driver that a new bus has appeared. You should avoid
	* using this, it will be removed in a near future.
	*/
	int (*attach_adapter)(struct i2c_adapter *) __deprecated;
	/* Standard driver model interfaces */
	int (*probe)(struct i2c_client *, const struct i2c_device_id *);
	int (*remove)(struct i2c_client *);
	
	/* driver model interfaces that don't relate to enumeration */
	void (*shutdown)(struct i2c_client *);
	int (*suspend)(struct i2c_client *, pm_message_t mesg);
	int (*resume)(struct i2c_client *);
	
	/* Alert callback, for example for the SMBus alert protocol.
	* The format and meaning of the data value depends on the protocol.
	* For the SMBus alert protocol, there is a single bit of data passed
	* as the alert response's low bit ("event flag").
	*/
	void (*alert)(struct i2c_client *, unsigned int data);
	
	/* a ioctl like command that can be used to perform specific functions
	* with the device.
	*/
	int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);
	
	struct device_driver driver;
	const struct i2c_device_id *id_table;
	
	/* Device detection callback for automatic device creation */
	int (*detect)(struct i2c_client *, struct i2c_board_info *);
	const unsigned short *address_list;
	struct list_head clients;
};
struct i2c_client {
	unsigned short flags; /* div., see below */
	unsigned short addr; /* chip address - NOTE: 7bit */
	/* addresses are stored in the */
	/* _LOWER_ 7 bits */
	char name[I2C_NAME_SIZE];
	struct i2c_adapter *adapter; /* the adapter we sit on */
	struct device dev; /* the device structure */
	int irq; /* irq issued by device */
	struct list_head detected;
};

int i2c_register_driver(struct module *owner, struct i2c_driver *driver);
void i2c_del_driver(struct i2c_driver *driver);
#define i2c_add_driver(driver) i2c_register_driver(THIS_MODULE, driver)
```
i2c_client对应于真实的物理设备，每个I2C设备都需要一个i2c_client来描述。一个i2c_driver可以支持多个同类型的i2c_client。i2c_driver对应于一套驱动回调函数，struct i2c_device_id形式的id_table是该驱动所支持的I2C设备的ID表。

由于一个I2C控制器可以连接多个I2C设备，所以一个i2c_adpater也可以被多个i2c_client依附。
```c
static struct i2c_driver yyy_driver = {
	.driver = {
		.name = "yyy",
	} ,
	.probe = yyy_probe,
	.remove = yyy_remove,
	.id_table = yyy_id,
};

static int __init yyy_init(void)
{
	return i2c_add_driver(&yyy_driver);
}
module_initcall(yyy_init);

static void __exit yyy_exit(void)
{
	i2c_del_driver(&yyy_driver);
}
module_exit(yyy_exit);

static ssize_t at24_eeprom_read(struct at24_data *at24, char *buf,unsigned offset, size_t count)
{
	struct i2c_msg msg[2];
	/* 第一条消息是写消息 */
	msg[0].addr = client->addr;
	msg[0].flags = 0;
	msg[0].len = 1;
	msg[0].buf = &offs;
	/* 第二条消息是读消息 */
	msg[1].addr = client->addr;
	msg[1].flags = I2C_M_RD;
	msg[1].len = sizeof(buf);
	msg[1].buf = &buf[0];
	i2c_transfer(client->adapter, msg, 2);
	...
}
```
设备驱动中使用下列函数向i2c控制器驱动传递数据：
```c
int i2c_transfer(struct i2c_adapter * adap, struct i2c_msg *msgs, int num);
int i2c_master_send(struct i2c_client *client,const char *buf ,int count);
int i2c_master_recv(struct i2c_client *client, char *buf ,int count);
```
i2c_transfer（）函数只是寻找到与i2c_adapter对应的i2c_algorithm，并使用i2c_algorithm的master_xfer（）函数真正驱动硬件传输流程。

## i2c-dev.c
i2c-dev.c文件可以被看作是一个I 2C设备驱动，不过它实现的i2c_client是虚拟。主要是为了便于从用户空间操作I2C控制器。i2c-dev.c实现了I2C控制器设备文件的功能，每一个I2C控制器都被分配一个设备文件。通过控制器设备文件访问设备时的主设备号都为89，次设备号为0~255。应用层可以借用这些接口访问挂接在控制器上的I2C设备的存储空间或寄存器，并控制I2C设备的工作方式。i2c-dev.c的主体是“i2c_driver成员函数+字符设备驱动”。

read（）和write（）分别调用I2C核心的i2c_master_recv（）和i2c_master_send（）函数来构造一条I2C消息并引发适配器Algorithm通信函数的调用，以完成消息传输。一个传输周期只有一条消息，不具备太强的通用性。

ioctl函数的I2C_RDWR命令支持自己构造i2c_msg数组，一个读写周期可以传递多个消息。常用的IOCTL命令包括I2C_SLAVE（设置从设备地址）、I2C_RETRIES（没有收到设备ACK情况下的重试次数，默认为1）、I2C_TIMEOUT（超时）以及I2C_RDWR(读写）。
