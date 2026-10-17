---
title: 08. linux阻塞io、非阻塞io、异步io
---

# 阻塞io与等待队列
```c
// 声明和初始等待队列头
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
DECLARE_WAIT_QUEUE_HEAD (name) 
// 定义等待队列元素，并绑定到指定进程
DECLARE_WAITQUEUE(name, tsk)
// 添加/移除队列元素
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
// 下列宏检查condition是否满足，阻塞并等待被唤醒，唤醒时检查condition是否满足，不满足继续阻塞
// 只需声明等待队列头，然后直接调用下列宏
wait_event(queue, condition)
wait_event_interruptible(queue, condition)
wait_event_timeout(queue, condition, timeout)
wait_event_interruptible_timeout(queue, condition, timeout)
// 唤醒指定队列中元素绑定的进程
void wake_up(wait_queue_head_t *queue);
void wake_up_interruptible(wait_queue_head_t *queue);
```
![Pasted image 20251109123130](assets/Pasted%20image%2020251109123130.png)

# 非阻塞io与select,poll,epoll轮询
```c
// app设置非阻塞flag
oflags = fcntl(FILENO, F_GETFL);
fcntl(FILENO, F_SETFL, oflags | O_NONBLOCK);
```
使用非阻塞I/O的应用程序通常会使用select/poll/epoll系统调用查询是否可对设备进行无阻塞的访问。系统调用最终会使设备驱动中的poll函数被执行。时序如下：
```c
用户进程调用 select/poll/epoll系统调用
	 ↓
内核调用驱动的 poll 方法
	 ↓
驱动调用 poll_wait()注册等待队列，并判断是否有事件
	 ↓
内核将进程加入等待队列
	 ↓  
无事件时用户进程在内核中休眠等待
	 ↓
设备事件发生
	 ↓
驱动调用 wake_up_interruptible()
	 ↓
内核唤醒休眠的进程
	 ↓
内核再次调用驱动的 poll 方法，此时poll_wait不会再次注册等待队列
	 ↓  
驱动返回事件掩码 (如 POLLIN)
	 ↓
select/poll/epoll 系统调用返回，用户进程处理事件
```
### select/poll/epoll函数 与 I/O 模式的组合

| 组合方式                | poll 超时  | fd 模式 | 行为特点                   |
| ------------------- | -------- | ----- | ---------------------- |
| `poll(fds, n, -1)`  | 阻塞直到事件唤醒 | 阻塞    | 传统阻塞模式，阻塞一个fd，剩下的都没法处理 |
| `poll(fds, n, -1)`  | 阻塞直到事件唤醒 | 非阻塞   | 事件驱动，read不阻塞           |
| `poll(fds, n, 0)`   | 不阻塞      | 阻塞    | 传统阻塞模式，阻塞一个fd，剩下的都没法处理 |
| `poll(fds, n, 0)`   | 不阻塞      | 非阻塞   | 真正的非阻塞轮询，忙轮询，浪费CPU     |
| `poll(fds, n, 100)` | 100ms超时  | 非阻塞   | 事件驱动+超时处理              |
如上所示，正确的事件驱动io组合只有两个。

##  file_operations中的poll回调函数
```c
unsigned int(*poll)(struct file * filp, struct poll_table* wait);
```
此回调应完成如下两项工作：
1）对可能引起设备文件状态变化的等待队列调用poll_wait（）函数，将对应的等待队列头部添加到poll_table中。
2）返回设备资源的可获取状态，即POLLIN、POLLOUT、POLLPRI、POLLERR、POLLNVAL等宏的“位或”结果。
```c
static unsigned int xxx_poll(struct file *filp, struct poll_table *wait)
{
	unsigned int mask = 0;
	struct xxx_dev *dev = filp->private_data; /* 获得设备结构体指针*/
	
	...
	poll_wait(filp, &dev->r_wait, wait); /* 加入读等待队列 */
	poll_wait(filp, &dev->w_wait, wait); /* 加入写等待队列 */
	
	if (...) /* 可读 */
		mask |= POLLIN | POLLRDNORM; /* 标示数据可获得（对用户可读）*/

	if (...) /* 可写 */
		mask |= POLLOUT | POLLWRNORM; /* 标示数据可写入*/
	...
	return mask;
}
static ssize_t xxx_read(struct file *filp, const char __user *buf, size_t count,loff_t *f_pos){
	if(no_data() && filp->f_flags & O_NONBLOCK)
		return -EAGAIN;
	if((filp->f_flags & O_NONBLOCK) == 0)
		wait_event_interruptible(dev->r_wait, hava_data());
	read_size = read_data();
	return read_size;
}
```
### poll_wait
```c
void poll_wait(struct file *filp, wait_queue_heat_t *queue, poll_table * wait);
```
不阻塞，在 驱动的 `poll` 方法中调用，仅注册等待队列，事件发生后需要wake_up_interruptible唤醒进程。虽然poll_wait不阻塞，但是用户空间的poll函数会阻塞，poll函数检查完所有驱动的poll函数后，如果没有事件发生，则将用户进程置于睡眠状态。

# 异步io与信号
在Linux中，异步通知使用信号来实现(详见：`man 7 signal`)。除了阻塞，非阻塞轮询，应用程序还可以注册信号回调函数，在事件发生后回调函数被调用以进行处理，信号相对于应用程序类似中断相对于cpu。
![Pasted image 20251213174311](assets/Pasted%20image%2020251213174311.png)
用户空间为了处理异步SIGIO信号需要进行三步操作：
- 通过F_SETOWN IO控制命令设置设备文件的拥有者为本进程，这样从设备驱动发出的SIGIO信号才能被本进程接收到。
- 通过F_SETFL IO控制命令设置设备文件以支持FASYNC，即异步通知模式。
- 通过signal函数注册信号处理函数。
```c
#include <fcntl.h>
#include <signal.h>
#define MAX_LEN 100
void input_handler(int num)
{
	char data[MAX_LEN];
	int len;
	
	/* 读取并输出STDIN_FILENO上的输入 */
	len = read(STDIN_FILENO, &data, MAX_LEN);
	data[len] = 0;
	printf("input available:%s\n",data);
}

main()
{
	int oflags;
	
	/* 启动SIGIO异步信号机制 */
	signal(SIGIO, input_handler);
	fcntl(FILENO, F_SETOWN, getpid());
	oflags = fcntl(FILENO, F_GETFL);
	fcntl(FILENO, F_SETFL, oflags | FASYNC);
	
	...
}
```

设备驱动程序中为了支持异步通知机制，也涉及3项工作，与应用空间的3项工作对应：
- 支持F_SETOWN命令，能在这个控制命令处理中设置filp->f_owner为对应进程ID。此项工作已由内核完成，设备驱动无须处理。
- 支持F_SETFL命令的处理，每当FASYNC标志改变时，驱动程序中的fasync（）函数将得以执行。因此，驱动中应该实现file_operations结构体的fasync（）函数。
- 在设备资源可获得时，调用kill_fasync（）函数激发相应的信号。
```c
struct xxx_dev {
	struct cdev cdev; /* cdev结构体*/
	...
	struct fasync_struct *async_queue; /* 异步结构体指针 */
};
static int xxx_fasync(int fd, struct file *filp, int mode)
{
	struct xxx_dev *dev = filp->private_data;
	return fasync_helper(fd, filp, mode, &dev->async_queue);
}
// 在设备资源可以获得时，调用kill_fasync（）释放SIGIO信号。在可读时，第3个参数设置为POLL_IN，在可写时，第3个参数设置为POLL_OUT。
static ssize_t xxx_write(struct file *filp, const char __user *buf, size_t count,loff_t *f_pos)
{
	struct xxx_dev *dev = filp->private_data;
	...
	/* 产生异步读信号 */
	if (dev->async_queue)
		kill_fasync(&dev->async_queue, SIGIO, POLL_IN);
	...
}
static int xxx_release(struct inode *inode, struct file *filp)
{
	/* 将文件从异步通知列表中删除 */
	xxx_fasync(-1, filp, 0);
	...
	return 0;
}
static struct file_operations fops = {
	.write = xxx_write,
	.fasync = xxx_fasync,
	.release = xxx_release,
}
```

# glibc提供的AIO和linux内核提供的AIO
## glibc实现的AIO接口
Linux的AIO有多种实现，其中一种实现是在用户空间的glibc库中实现的，它本质上是借用了多线程模型，用开启新的线程以同步的方法来做I/O，新的AIO辅助线程与发起AIO的线程以pthread_cond_signal（）的形式进行线程间的同步。详细信息参考：
https://www.gnu.org/software/libc/manual/html_node/Asynchronous-I_002fO.html
glibc的libaio库包装了linux实现的AIO接口，虽然内核中的AIO接口已被io_uring取代，但glibc库中的libaio接口依然未发生变化，主要接口有：
```c
int io_setup(int maxevents, io_context_t *ctxp);
int io_destroy(io_context_t ctx);
int io_submit(io_context_t ctx, long nr, struct iocb *ios[]);
int io_cancel(io_context_t ctx, struct iocb *iocb, struct io_event *evt);
int io_getevents(io_context_t ctx_id, long min_nr, long nr, struct io_event *events,
struct timespec *timeout);
void io_set_callback(struct iocb *iocb, io_callback_t cb);
void io_prep_pwrite(struct iocb *iocb, int fd, void *buf, size_t count, long long offset);
void io_prep_pread(struct iocb *iocb, int fd, void *buf, size_t count, long long offset);
void io_prep_pwritev(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt,
long long offset);
void io_prep_preadv(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt,
long long offset);
```
AIO的读写请求都用io_submit（）下发。下发前通过io_prep_pwrite（）和io_prep_pread（）生成iocb的结构体，作为io_submit（）的参数。这个结构体指定了读写类型、起始地址、长度和设备标志符等信息。读写请求下发之后，使用io_getevents（）函数等待I/O完成事件。io_set_callback（）则可设置一个AIO完成的回调函数。

更好的方式是直接使用[liburing](https://github.com/axboe/liburing)库。

# linux内核实现的AIO
linux内核中的AIO演变时间线：
- 4.1版本之前的内核在file_operations结构中提供了实现的AIO接口(aio_read, aio_write, aio_fsync)
- 4.1~4.3版本开始弃用file_operations中的AIO接口(aio_read, aio_write, aio_fsync)
- 4.3版本完全移除了这3个接口
- 5.1版本引入了io_uring，使用file_operations中的(read_iter, write_iter)作为接口
## AIO与设备驱动
旧的AIO接口(aio_read, aio_write, aio_fsync)不再讨论，新的io_uring使用file_operations中的(read_iter, write_iter)作为接口，统一了同步和异步接口(直接实现read_iter，可以不实现read)：
```c
// 使用 read_iter 统一处理
static ssize_t my_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    struct file *filp = iocb->ki_filp;
    struct my_device *dev = filp->private_data;
    if (iocb->ki_flags & IOCB_DIRECT) {
        return my_direct_io(iocb, iter);  // 直接IO
    } else {
        return my_buffered_io(iocb, iter); // 缓冲IO
    }
}

// 同时处理同步和异步
static const struct file_operations my_fops = {
    .read_iter = my_read_iter,
    .write_iter = my_write_iter,
    // 不再需要单独的 .read, .write, .aio_read, .aio_write
};
```
传统向量io需要手动遍历iovec数组，新的迭代器接口Iov_iter具有更好的向量支持，iov_iter支持多种数据源，可以统一处理用户空间缓冲区 (iovec)、内核空间缓冲区 (kvec)  、双向管道缓冲区、其他特殊缓冲区类型：
```c
// read_iter 使用更先进的 iov_iter
static ssize_t my_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    // iov_iter 自动处理所有类型的向量
    while (iov_iter_count(iter) > 0) {
        size_t len = process_chunk(iter);
        iov_iter_advance(iter, len);
    }
    // 同样的代码处理所有情况，驱动不需要关心数据的具体来源
    copy_to_iter(device_data, data_len, iter);
    copy_from_iter(device_data, data_len, iter);
}
```

# io_uring
https://arthurchiao.art/blog/intro-to-io-uring-zh/