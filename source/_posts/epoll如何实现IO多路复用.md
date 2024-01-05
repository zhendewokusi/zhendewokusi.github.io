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
		sock_release(newsock);
    
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

	// 1. ep_alloc
	error = ep_alloc(&ep);
	// 错误处理
    // ...

	// 2. 创建匿名的文件对象以及文件描述符
	error = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	return error;
}
```

### 1. ep_alloc

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
- **wq**：软中断数据就绪的时候会通过wq来找到阻塞在epoll对象上的用户**进程**。等待队列链表用于管理等待事件发生的进程。
- **rbr**：红黑树用于存储被监控的文件描述符（fd）的数据结构及其状态信息。
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
	// ep的初始化操作
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

### 2. 创建匿名的文件对象以及文件描述符

匿名文件对象是内核用于表示一些临时或者有特殊用途的文件对象，其是在内存中动态创建和管理的，常见用于进程间通信（比如`pipe`、`fifo`）、临时文件、内核模块之间的通信以及一些特殊的操作。通过将文件挂接在单个 inode 上来创建新文件。这对于不需要拥有完整 inode 即可正确运行的文件非常有用。使用 anon_inode_getfd() 创建的所有文件将共享一个 inode，从而节省内存并避免文件/inode/dentry 设置的代码重复。将`epoll`和文件描述符关联起来，内核也更方便管理和处理被监视的文件描述符的事件。

```c
int anon_inode_getfd(const char *name, const struct file_operations *fops,
		     void *priv, int flags)
{
	int error, fd;
	struct file *file;
	// 获取一个未被使用的文件描述符
	error = get_unused_fd_flags(flags);
	if (error < 0)
		return error;
	fd = error;
	// 创建一个匿名文件对象
	file = anon_inode_getfile(name, fops, priv, flags);
	// 创建文件对象发生错误的处理
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto err_put_unused_fd;
	}
	// 将该匿名文件对象和文件描述符关联起来
	fd_install(fd, file);

	return fd;

err_put_unused_fd:
	put_unused_fd(fd);
	return error;
}
EXPORT_SYMBOL_GPL(anon_inode_getfd);
```

总结epoll_create函数所做的事：调用epoll_create后，在内核中分配一个eventpoll结构和代表epoll文件的file结构，并且将这两个结构关联在一块，同时，返回一个也与file结构相关联的epoll文件描述符fd。当应用程序操作epoll时，需要传入一个epoll文件描述符fd，内核根据这个fd，找到epoll的file结构，然后通过file，获取之前epoll_create申请eventpoll结构变量，epoll相关的重要信息都存储在这个结构里面。接下来，所有epoll接口函数的操作，都是在eventpoll结构变量上进行的。

所以，epoll_create的作用就是为进程在内核中建立一个从epoll文件描述符到eventpoll结构变量的通道。

## epoll_ctl 的实现
这里是理解epoll的核心区域（这么久了，终于到了....）。
首先epoll_ctl的作用是添加、修改、删除文件的监听事件，[内核代码](https://elixir.bootlin.com/linux/v2.6.39.4/source/fs/eventpoll.c#L1347)：
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
    // 1. 根据操作判断是否拷贝到内核以及获取 file 对象
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;
    //#define	EBADF		 9	/* Bad file number */
	error = -EBADF;
	// 通过文件描述符获取 file对象
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

	// 2. 循环引用的处理
	if (unlikely(is_file_epoll(tfile) && op == EPOLL_CTL_ADD)) {
		mutex_lock(&epmutex);
		did_lock_epmutex = 1;
		error = -ELOOP;
		if (ep_loop_check(ep, tfile) != 0)
			goto error_tgt_fput;
	}


	mutex_lock(&ep->mtx);
	// 3. ep_find
	epi = ep_find(ep, tfile, fd);

	error = -EINVAL;
	// 4. 
	switch (op) {(!is_file_epoll(f.file));
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= POLLERR | POLLHUP;
			error = nsert(ep, &epds, tfile, fd);
		} else
			error = -EEXIST;
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;flags
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
### 1. 根据操作判断是否拷贝到内核以及获取 file 对象

这里的`op`是对`epoll`操作动作（添加、删除、修改），`ep_op_has_event(op)`是判断是否是删除操作，如果`op != EPOLL_CTL_DEL 为 true`时，则需要调用`copy_from_user()`将`event`事件拷贝到内核的`epds`变量中。因为只有删除操作不需要内核使用进程传入的`event`事件。

```c
static inline int ep_op_has_event(int op)
{
	return op != EPOLL_CTL_DEL;
}
```

接下来调用两次`fget`分别获取epoll文件和连接套接字文件的file结构变量。

接下来就是对参数的一些检查，出现如下情况，就可以认为传入的参数有问题，直接返回出错：

1. 目标文件不支持poll操作(!tf.file->f_op->poll)；
2. 监听的目标文件就是epoll文件本身(f.file == tf.file)；
3. 用户传入的epoll文件(epfd代表的文件）并不是一个真正的epoll的文件(!is_file_epoll(f.file));

### 2. 循环引用的处理

这里会有一个问题，假设有两个epoll文件描述符 A 和 B，且A已经插入到B的文件描述符中，而某个时刻B直接或者间接插入到A中，这就形成闭环，导致事件发生时递归处理，从来形成文件的事件循环，这就是循环引用。它会导致死锁或者其他的问题。这里为了解决循环引用，代码中是下面的措施：
```c
	if (unlikely(is_file_epoll(tfile) && op == EPOLL_CTL_ADD)) {
		mutex_lock(&epmutex);	
		did_lock_epmutex = 1;
		error = -ELOOP;
		if (ep_loop_check(ep, tfile) != 0)
			goto error_tgt_fput;
	}
```
这里先判断插入的文件描述符是否是epoll文件描述符，是否是`EPOLL_CTL_ADD`操作，条件满足时会先`mutex_lock(&epmutex);`获取全局锁防止处理时发生竞争，接下来判断是否存在闭环`ep_loop_check(ep, tfile) != 0`，如果存在进入对应错误处理。

### 3. ep_find
 <!-- Try to lookup the file inside our RB tree, Since we grabbed "mtx" above, we can be sure to be able to use the item looked up by ep_find() till we release the mutex. -->

在ep里面，维护着一个红黑树`rbr`，每次添加注册事件时，都会申请一个epitem结构的变量表示事件的监听项，然后插入ep的红黑树里面。在epoll_ctl里面，会调用ep_find函数从ep的红黑树里面查找目标文件表示的监听项，返回的监听项可能为空。而`epoll_filefd`[结构体](https://elixir.bootlin.com/linux/v2.6.39.4/source/fs/eventpoll.c#L93)中只有两个成员:file结构体和其对应的fd。

```c
static struct epitem *ep_find(struct eventpoll *ep, struct file *file, int fd)
{
	int kcmp;
	struct rb_node *rbp;
	struct epitem *epi, *epir = NULL;
	struct epoll_filefd ffd;
	//epi 用于遍历红黑树节点，epir 用于存储最终找到的 epitem 结构体
	ep_set_ffd(&ffd, file, fd);
	// 遍历红黑树查找其对应的 epitem
	for (rbp = ep->rbr.rb_node; rbp; ) {
		epi = rb_entry(rbp, struct epitem, rbn);
		kcmp = ep_cmp_ffd(&ffd, &epi->ffd);
		if (kcmp > 0)
			rbp = rbp->rb_right;
		else if (kcmp < 0)
			rbp = rbp->rb_left;
		else {
			epir = epi;
			break;
		}
	}

	return epir;
}
```
### 4. EPOLL_CTL_ADD情况

接下来switch这块区域的代码就是整个epoll_ctl函数的核心，对op进行switch出来的有添加(EPOLL_CTL_ADD)、删除(EPOLL_CTL_DEL)和修改(EPOLL_CTL_MOD)三种情况，这里我以添加为例讲解，其他两种情况类似，知道了如何添加监听事件，其他删除和修改监听事件都可以举一反三。
```c
	case EPOLL_CTL_ADD:
		if (!epi) {
			// epds 是 epollevent
			epds.events |= POLLERR | POLLHUP;
			error = ep_insert(ep, &epds, tfile, fd);
		} else
			error = -EEXIST;
		break;
```

为目标文件添加监控事件时，首先要保证当前ep里面还没有对该目标文件进行监听，如果存在(epi不为空)，就返回-EEXIST错误。否则说明参数正常，然后先默认设置对目标文件的POLLERR和POLLHUP监听事件，然后调用ep_insert函数，将对目标文件的监听事件插入到ep维护的红黑树里面,下面是ep_insert的源码：
```c
// fs.eventpoll.c
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
		     struct file *tfile, int fd)
{
	int error, revents, pwake = 0;
	unsigned long flags;
	long user_watches;
	struct epitem *epi;
	struct ep_pqueue epq;
	
	// 检查 epoll 实例中的监视数是否超过上限
	user_watches = atomic_long_read(&ep->user->epoll_watches);
	if (unlikely(user_watches >= max_user_watches))
		return -ENOSPC;
	
	// 分配 epitem 项结构体
	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
		return -ENOMEM;

	// epi 初始化
	INIT_LIST_HEAD(&epi->rdllink);
	INIT_LIST_HEAD(&epi->fllink);
	INIT_LIST_HEAD(&epi->pwqlist);
	epi->ep = ep;
	ep_set_ffd(&epi->ffd, tfile, fd);
	epi->event = *event;
	epi->nwait = 0;
	epi->next = EP_UNACTIVE_PTR;

	epq.epi = epi;
	// 初始化回调函数
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

	//  将回调函数 ep_ptable_queue_proc 与 tfile 文件关联起来，以及获取当前文件的事件位
	revents = tfile->f_op->poll(tfile, &epq.pt);

	error = -ENOMEM;
	if (epi->nwait < 0)
		goto error_unregister;

	// 调用list_add_tail_rcu将当前监听项添加到目标文件的f_ep_links链表里面
	// 该链表是目标文件的epoll钩子链表，所有对该目标文件进行监听的监听项都会加入到该链表里面。
	spin_lock(&tfile->f_lock);
	list_add_tail(&epi->fllink, &tfile->f_ep_links);
	spin_unlock(&tfile->f_lock);


	// 将epi监听项添加到ep维护的红黑树里面
	ep_rbtree_insert(ep, epi);

	spin_lock_irqsave(&ep->lock, flags);

	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
		list_add_tail(&epi->rdllink, &ep->rdllist);

		if (waitqueue_active(&ep->wq))
			wake_up_locked(&ep->wq);
		if (waitqueue_active(&ep->poll_wait))
			pwake++;
	}

	spin_unlock_irqrestore(&ep->lock, flags);

	atomic_long_inc(&ep->user->epoll_watches);

	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return 0;

error_unregister:
	ep_unregister_pollwait(ep, epi);

	spin_lock_irqsave(&ep->lock, flags);
	if (ep_is_linked(&epi->rdllink))
		list_del_init(&epi->rdllink);
	spin_unlock_irqrestore(&ep->lock, flags);

	kmem_cache_free(epi_cache, epi);

	return error;
}
```

在调用epoll_ctl时，可能会产生相关进程需要监听的事件，如果有监听的事件产生，(revents & event->events 为 true)，并且目标文件相关的监听项没有链接到ep的准备链表rdlist里面的话，就将该监听项添加到ep的rdlist准备链表里面，rdlist链接的是该epoll描述符监听的所有已经就绪的目标文件的监听项。并且，如果有任务在等待产生事件时，就调用wake_up_locked函数唤醒所有正在等待的任务，处理相应的事件。当进程调用epoll_wait时，该进程就出现在ep的wq等待队列里面。


--------------------------------
[回调函数](https://elixir.bootlin.com/linux/v2.6.39.4/source/include/linux/rcupdate.h#L60)
```c
#include <linux/rcupdate.h>

struct my_data {
    // 数据结构的定义
    int value;
    struct rcu_head rcu;  // 包含在数据结构中的 struct rcu_head
};

// 初始化数据结构并将其插入 RCU 队列
void enqueue_for_removal(struct my_data *data) {
    call_rcu(&data->rcu, my_data_free_callback);
}

// 实际的释放回调函数
void my_data_free_callback(struct rcu_head *head) {
    struct my_data *data = container_of(head, struct my_data, rcu);
    kfree(data);
}

```

[sk_data_ready](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/ipv4/tcp_input.c#L5401)
[sock_def_readable](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/core/sock.c#L1904)