---
title: epoll如何实现IO多路复用
date: 2023-12-20 19:21:24
tags: [epoll]
---

大一暑假完成了一个C++在线聊天室，对epoll的应用仅仅在于那三个函数：
- epoll_create
- epoll_ctl
- epoll_wait


看看epoll的源码实现，理解epoll是如何实现IO多路复用的。这里我看的是linux2.6.39.4版本的源码，在[在线网站](https://elixir.bootlin.com/linux/v2.6.39.4/source)可以查看。如果有错误敬请斧正。

## accept 系统调用创建新的 socket 对象

```c
// net/socket.c:1478
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen, int, flags)
{
	struct socket *sock, *newsock;
	struct file *newfile;
	int err, len, newfd, fput_needed;
	struct sockaddr_storage address;
    // 0. 关于 flags 的处理
	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
		return -EINVAL;

	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;
    // 1. 根据 fd 查找到监听的 socket
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (!sock)
		goto out;

	err = -ENFILE;
    // 2. alloc 一个新的 socket 对象
	newsock = sock_alloc();
	if (!newsock)
		goto out_put;
    // 初始化新的socket部分信息
	newsock->type = sock->type;
	newsock->ops = sock->ops;

	/*
	 * We don't need try_module_get here, as the listening socket (sock)
	 * has the protocol module (sock->ops->owner) held.
	 */
     // 在套接字创建的过程中，为了确保模块在使用期间不会被卸载，需要增加模块的引用计数。
	__module_get(newsock->ops->owner);
    // 3. 为新的 socket 分配一个新的 file 对象
	newfd = sock_alloc_file(newsock, &newfile, flags);
	if (unlikely(newfd < 0)) {
		err = newfd;
		sock_release(newsock);敬请斧正
    
	err = security_socket_accept(sock, newsock);
	if (err)
		goto out_fd;
    // 4. 接收连接
	err = sock->ops->accept(sock, newsock, sock->file->f_flags);
	if (err < 0)
		goto out_fd;

	if (upeer_sockaddr) {
		if (newsock->ops->getname(newsock, (struct sockaddr *)&address,
					  &len, 2) < 0) {
			err = -ECONNABORTED;
			goto out_fd;
		}
		err = move_addr_to_user((struct sockaddr *)&address,
					len, upeer_sockaddr, upeer_addrlen);
		if (err < 0)
			goto out_fd;
	}

	/* File flags are not inherited via accept() unlike another OSes. */
    
    // 5. 将新的文件添加到当前进程的打开文件列表
	fd_install(newfd, newfile);
	err = newfd;

out_put:
	fput_light(sock->file, fput_needed);
out:
	return err;
out_fd:
	fput(newfile);
	put_unused_fd(newfd);
	goto out_put;
}
}

SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen)
{
	return sys_accept4(fd, upeer_sockaddr, upeer_addrlen, 0);
}
```
### 0. 关于 flags 的处理

可以看到这里`accept`和`accept4`的区别仅仅是多了一个`flags`的参数，如果想了解这些`flags`的设置可以自行搜索。如果使用`accept`，本质上运行的是`flags`参数为0的`accept4`。

### 1. 根据 fd 查找到监听的 socket

```c
// include/asm-generic/error-base.h
// 可以看到这个宏对应没有找到套接字对象
#define	EBADF		 9	/* Bad file number */

// net/socket.c:450
static struct socket *sockfd_lookup_light(int fd, int *err, int *fput_needed)
{
	struct file *file;
	struct socket *sock;

	*err = -EBADF;
	file = fget_light(fd, fput_needed);
	if (file) {
		sock = sock_from_file(file, err);
		if (sock)
			return sock;
		fput_light(file, *fput_needed);
	}
	return NULL;
}
```
这里还有另一个参数`fput_needed`，该参数是在`fget_light`内被初始化的，感兴趣的读者可以去看看,路径是`/fs/file_table.c`第331行，简单说一下其作用：
- 如果 fput_needed 的值为 1，表示文件引用计数已经成功增加，调用者在使用完文件后需要调用`fput`函数来减少引用计数。
- 如果 fput_needed 的值为 0，表示文件引用计数未增加，可能由于并发操作中文件已被释放，调用者无需调用`fput`。

套接字分为两类：监听套接字和连接套接字。监听套接字是使用`socket()`创建一个套接字，`bind()`将其绑定到一个特定的本地地址和端口，`listen()`开始监听连接请求。连接套接字是使用`accept()`来创建与特定客户进行通信。而创建连接套接字的时候会使用监听套接字的部分设置（比如地址簇A、套接字类型、协议类型等等），当然连接套接字也有自己的一些设置，比如新的本地端口号等等。这里`sockfd_lookup_light`函数的作用是通过`fd`来找到监听套接字的`socket`结构体指针。准备后面的部分设置共享。

### 2. alloc 一个新的 socket 对象

这里首先调用`sock_alloc`函数申请一个新的`socket`对象。然后将监听套接字的`socket`对象的`type`和协议操作函数集合[ops](https://elixir.bootlin.com/linux/v2.6.39.4/source/include/linux/net.h#L160)赋给新`socket`对象。这里的`ops`里面主要是协议族，内核模块，以及一些需要的函数，感兴趣的可以去看看。需要注意的是这里的`ops`是一个指针，并不是将其整体复制了一份。

```c
struct socket {
	socket_state		state;
	kmemcheck_bitfield_begin(type);
	short			type;
	kmemcheck_bitfield_end(type);
	unsigned long		flags;
	struct socket_wq __rcu	*wq;
	struct file		*file;  // file descriptor
	struct sock		*sk;    // 核心成员
	const struct proto_ops	*ops;
};
```

### 3. 为新的 socket 分配一个新的 file 对象

这里的`file`对象其实就表示打开的文件对象，也就是文件描述符。使得连接套接字能通过文件描述符进行访问和操作。对其感兴趣可以点击该[链接](https://elixir.bootlin.com/linux/v2.6.39.4/source/include/linux/fs.h#L933)查看file结构体的具体细节。

因为连接套接字的`socket`对象的`file`指针还是空的。接下来调用`sock_alloc_file`函数来申请并且初始化该对象。并且将其设置在`sock->file`对象。具体代码点[链接](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/socket.c#L389)。

新的socket对象->file->f_op->poll[函数](https://elixir.bootlin.com/linux/v2.6.39.4/source/include/linux/fs.h#L1545)指向的是sock_poll，之后会用到的。

### 4. 接收连接

这里介绍另一个`socket`对象中的核心成员`sock`。该[结构体](https://elixir.bootlin.com/linux/v2.6.39.4/source/include/net/sock.h#L238)非常大，其中发送队列、接收数据包的队列、错误队列、缓存锁、异步等待的队列、数据包的哈希值，数量等核心数据结构都在这里。
```c
    // 安全性检查
	err = security_socket_accept(sock, newsock);
	if (err)
		goto out_fd;

	err = sock->ops->accept(sock, newsock, sock->file->f_flags);
	if (err < 0)
		goto out_fd;
```
这里的[accept](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/tipc/socket.c#L1491)对应的是`inet-accept`。它会从握手队列中直接获取创建好的`sock`。

### 5. 将新的文件添加到当前进程的打开文件列表

当 file、socket、sock 等关键内核对象创建完毕以后，剩下要做的一件事情就是把它挂到当前进程的打开文件列表中就行了。
```c
// fs/file.c
void fd_install(unsigned int fd, struct file *file)
{
    __fd_install(current->files, fd, file);
}

void __fd_install(struct files_struct *files, unsigned int fd,
        struct file *file)
{
    fdt = files_fdtable(files);
    BUG_ON(fdt->fd[fd] != NULL);
    rcu_assign_pointer(fdt->fd[fd], file);
}
```
<!-- [socket()](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/socket.c#L1294)
[socket_create](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/socket.c#L1270) -->

## epoll_create 的实现

用户进程在调用`epoll_create`时，内核会创建一个`eventpoll`对象。将其关联到当前进程的已打开文件列表中。`epoll_create`[源码](https://elixir.bootlin.com/linux/v2.6.39.4/source/fs/eventpoll.c#L1306)：
```c
// fs/eventpool.c
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
	int error;
	struct eventpoll *ep = NULL;
	// flags处理
    // ...
	error = ep_alloc(&ep);
	// 错误处理
    // ...
	return error;
}
```
`eventpoll`的[实现](https://elixir.bootlin.com/linux/v2.6.39.4/source/fs/eventpoll.c#L158)，这里我们只看主要的成员：
```c
// fs/eventpool.c
struct eventpoll {
    // 等待队列，链表
	wait_queue_head_t wq;
	// 接收就绪的描述符，链表
    struct list_head rdllist;
	// 红黑树
    struct rb_root rbr;
};
```
成员作用：
- **wq**：软中断数据就绪的时候会通过wq来找到阻塞在epoll对象上的用户进程。
- **rbr**：管理用户进程下添加进来的所有socket连接。
- **rdllist**：当有连接就绪时，会将就绪的连接放在该链表中。这样就不用遍历整棵树。
然后在`ep_alloc`里面实现初始化工作，[源码](https://elixir.bootlin.com/linux/v2.6.39.4/source/fs/eventpoll.c#L748)：
```c
static int ep_alloc(struct eventpoll **pep)
{
	int error;
	struct user_struct *user;
	struct eventpoll *ep;

	user = get_current_user();
	error = -ENOMEM;
	ep = kzalloc(sizeof(*ep), GFP_KERNEL);
	if (unlikely(!ep))
		goto free_uid;

	spin_lock_init(&ep->lock);
	mutex_init(&ep->mtx);
	init_waitqueue_head(&ep->wq);
	init_waitqueue_head(&ep->poll_wait);
	INIT_LIST_HEAD(&ep->rdllist);
	ep->rbr = RB_ROOT;
	ep->ovflist = EP_UNACTIVE_PTR;
	ep->user = user;

	*pep = ep;

	return 0;

free_uid:
	free_uid(user);
	return error;
}
```

## epoll_ctl 的实现
(https://elixir.bootlin.com/linux/v2.6.39.4/source/fs/eventpoll.c#L1347)

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int did_lock_epmutex = 0;
	struct file *file, *tfile;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
    //#define	EFAULT		14	/* Bad address */
	error = -EFAULT;
    // 检查参数，并从用户空间复制事件的数据到内核空间
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;
    //#define	EBADF		 9	/* Bad file number */
	error = -EBADF;
	file = fget(epfd);
	if (!file)
		goto error_return;
	tfile = fget(fd);
	if (!tfile)
		goto error_fput;
    //#define	EPERM		 1	/* Operation not permitted */
	error = -EPERM;
	if (!tfile->f_op || !tfile->f_op->poll)
		goto error_tgt_fput;

    //#define	EINVAL		22	/* Invalid argument */
	error = -EINVAL;
	if (file == tfile || !is_file_epoll(file))
		goto error_tgt_fput;


	ep = file->private_data;


	if (unlikely(is_file_epoll(tfile) && op == EPOLL_CTL_ADD)) {
		mutex_lock(&epmutex);
		did_lock_epmutex = 1;
		error = -ELOOP;
		if (ep_loop_check(ep, tfile) != 0)
			goto error_tgt_fput;
	}


	mutex_lock(&ep->mtx);

	epi = ep_find(ep, tfile, fd);

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_insert(ep, &epds, tfile, fd);
		} else
			error = -EEXIST;
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_modify(ep, epi, &epds);
		} else
			error = -ENOENT;
		break;
	}
	mutex_unlock(&ep->mtx);

error_tgt_fput:
	if (unlikely(did_lock_epmutex))
		mutex_unlock(&epmutex);

	fput(tfile);
error_fput:
	fput(file);
error_return:

	return error;
}
```
