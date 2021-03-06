# epoll方式
## 主体流程
> 1. ### [socket](#1-socket)
> 2. ### [bind](#2-bind)
> 3. ### [listen](#3-listen)
> 4. ### [epoll_create](#4-epoll_create)
> 5. ### [epoll_ctl](#5-epoll_ctl)
> 6. ### [epoll_wait](#6-epoll_wait)
  

## 步骤详解
## 1. socket
````c
/*
 *  该函数用于打开一个socket套接字，返回值为一个文件描述符
 *	@domain: 通讯域. 通常是AF_INET
 *	@type: 通讯类型. 字节流为SOCK_STREAM，数据报为SOCK_DGRAM
 *	@protocol: 协议
 *	@return: 文件描述符，失败时返回-1
 */
int socket(int domain, int type, int protocol);
````
1. 将函数转换为系统调用(sys_call)号  

2. 从sys_call_table系统调用表中找到对应的系统调用

3. 调用sock_create(``net/socket.c``)构造一个套接字  

4. 继续调用__sock_create(current->nsproxy->net_ns, family , type, protocol, res, 0)，其中第一个参数是获取当前进程的namespace proxy，再根据namespace proxy获取其网络命名空间
  
5. 检查入参，如协议族、通讯类型是否符合取值范围  

6. 调用security_socket_create, 其本质是一个钩子，在socket创建前回调，该函数为LSM框架声明的一个函数(``security/lsm.c``)  

7. 调用sock_alloc开始创建socket struct，内部会构造一个inocde，该inode唯一标识当前socket的通信

8. 调用inet_create(``net/ipv4/af_inet.c``)创建一个inet的套接字(AF_INET)

9. 调用security_socket_post_create, 表示socket创建后置回调钩子。与6对应

10. 调用sock_map_fd将创建好的socket struct绑定到一个未使用的文件描述符fd上

11. 调用sock_alloc_file分配一个file struct, 并将其相关联。	``sock->file = file; file->private_data = sock;``
12. 调用stream_open将该文件描述符的f_mode标记为STREAM，表示同时可读写，与其他的文件描述符相反，其他文件描述符不能同时读和写

13. 将前面的到的文件描述符``fd`` 放置到``fdtable``(文件描述符表)  

14. 从内核态退出到用户态，完成本次``socket``函数调用 

## 2. bind
1. 系统调用__sys_bind(``net/socket.c``)进入bind逻辑

2. 调用sockfd_lookup_light，轻量级方式获取socket struct  

3. 调用``fdget(fd)``,根据传入的``int fd``获取具体的文件描述符结构体。其内部是调用``__fget_light``(``fs/file.c``)函数获得fd的

4. 根据当前进程上下文(``task_struct``)获取``files_sturct``, 再通过``files_sturct``得到``fdtable(include/linux/fdtable.h)``文件描述符缓存表，最终得到``file``后再转换为``unsigned long``, 再将long转换为具体的``fd struct``(``__to_fd(include/linux/file.h)``)  

5. 将``file``转为``socket strcut (sock_from_file)``  

6. 调用``move_addr_to_kernel``将用户态socket地址复制到内核态中

7. 回调``security_socket_bind``的钩子函数(``LSM``)

8. 将具体的``socket``绑定到指定的地址上(``net/ipv4/af_inet.c``)。
参考[What is an L3 Master Device](https://legacy.netdevconf.info/1.2/papers/ahern-what-is-l3mdev-paper.pdf)
````c
int inet_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
````

## 3. listen
1. 入参为(socket fd, backlog)执行系统调用__sys_listen(``net/socket.c``)进入``listen``逻辑

2. 调用sockfd_lookup_light，轻量级方式获取socket struct  

3. sockfd_lookup_light的逻辑与 [2.bind](#2-bind) 中介绍的一样  

4. 从socket结构体中取出对应的``somaxconn``,该值默认4096(``cat /proc/sys/net/core/somaxconn``)  

5. ``somaxconn``与传入的``backlog``对比，当传入的 backlog > somaxconn时，则使用somaxconn作为backlog，反之亦然  

6. 调用security_socket_listen(LSM回调),检查是否允许调用``listen``函数  

7. 调用 ``sock->ops->listen(sock, backlog)``,listen的实现位于``net/ipv4/af_inet.c中的inet_listen``,该函数的主要目的是修改``socket``的状态为``listening``  

8. 对sock上互斥锁并检查该socket的状态是否是``TCP_LISTEN``。如果如果不是``TCP_LISTEN``状态, 则开始检查``TFO(TCP fast open)``特性是否打开(``/proc/sys/net/ipv4/tcp_fastopen``)
````
/proc/sys/net/ipv4/tcp_fastopen
0：关闭
1：作为客户端使用Fast Open功能，默认值
2：作为服务端使用Fast Open功能
3：无论是客户端还是服务端都使用Fast Open功能
````

9. 调用inet_csk_listen_start函数

10. 为``inet_connection_sock``分配SYN_RECV队列``request_sock_queue``

11. 更改sock状态为``TCP_LISTEN``

12. 调用inet_csk_get_port(``net/ipv4/inet_connection_sock.c``)获取给定的socket的本地端口。如果端口为0, 则会随机分配一个奇数端口, 并为``connect()``保留偶数的端口

13. 将socket绑定到L3 Master device[What is an L3 Master Device](https://legacy.netdevconf.info/1.2/papers/ahern-what-is-l3mdev-paper.pdf)上,配置(``cat /proc/sys/net/ipv4/tcp_l3mdev_accept``)

14. 从``inet_bind_hashbucket``中查找tb(``inet_bind_bucket``), 如果找到了对应的tb(``inet_bind_bucket``),则跳转到``tb_found`` label代码片段
````c
struct inet_bind_hashbucket {
	spinlock_t		lock;
	struct hlist_head	chain;
};
````

15. 检查当前的socket reuse(TCP重用)状态, 如果是SK_FORCE_REUSE则立即跳转到``success``label代码块
````c
tb_not_found:
	tb = inet_bind_bucket_create(hinfo->bind_bucket_cachep,
				     net, head, port, l3mdev);
	if (!tb)
		goto fail_unlock;
tb_found:
	if (!hlist_empty(&tb->owners)) {
		if (sk->sk_reuse == SK_FORCE_REUSE)
			goto success;

		if ((tb->fastreuse > 0 && reuse) ||
		    sk_reuseport_match(tb, sk))
			goto success;
		if (inet_csk_bind_conflict(sk, tb, true, true))
			goto fail_unlock;
	}
success:
	inet_csk_update_fastreuse(tb, sk);

	if (!inet_csk(sk)->icsk_bind_hash)
		inet_bind_hash(sk, tb, port);
	WARN_ON(inet_csk(sk)->icsk_bind_hash != tb);
	ret = 0;

fail_unlock:
	spin_unlock_bh(&head->lock);
	return ret;
}
````

16. 成功后将当前的socket TX queue清空

17. 尝试计算当前socket的hash(``net/ipv4/inet_hashtables.c``),并从``listening_hash``取出``inet_listen_hashbucket``,获取inet_listen_hashbucket中的自旋锁,尝试将重用的port添加到当前socket中

18. 将当前socket、port进行hash并存放到``inet_listen_hashbucket``上

19. 将当前使用的socket port添加到``已使用表``中

## 4. epoll_create
1. 调用``epoll_create()``函数进入``do_epoll_create(fs/eventpoll.c)``

2. 调用``ep_alloc(fs/eventpoll.c)``函数创建``eventpoll``

3. 调用``get_current_user``获取当前用户

4. 调用``kzalloc``分配一段内存, 内存大小为``eventpoll``的大小(sizeof)

5. 初始化``eventpoll``的相关属性,如互斥锁、读写锁、等待队列、红黑树、所属用户

6. 调用``get_unused_fd_flags``获取未使用的文件描述符fd

7. 调用``anon_inode_getfile``函数获取对应的文件``file``

8. 将文件描述符``fd``与文件``file``绑定

9. 返回文件描述符``fd``

## 5. epoll_ctl
1. 调用系统调用SYSCALL_DEFINE4(``fs/eventpoll.c``)

2. 检查当前操作符是否合法，并将事件``EPOLLIN``从用户空间复制到内核空间,复制长度为``sizeof(struct epoll_event))``

3. 调用``do_epoll_ctl``

4. 调用``fget(epfd)``通过刚才创建epoll文件描述符的到的数值获取对应的``struct fd``

5. 调用``fdget(sockfd)``函数,将之前调用``socket``返回的sockfd传入,以此获取``struct fd``

6. 检查``sockfd``是否支持``poll``操作(``fd.file->f_op->poll``)

7. 检查是否允许 EPOLLWAKEUP

8. 检查当前event是否是独占事件``EPOLLEXCLUSIVE``, 如果是``EPOLL_CTL_MOD``或者``EPOLL_CTL_ADD``是不被允许的

9. 从``epoll fd``中取出其file的私有数据``struct eventpoll``, 并调用``epoll_mutex_lock(&ep->mtx, 0, nonblock)``获取``eventpoll``的互斥量并上锁
````c
struct eventpoll {
	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 */
	struct mutex mtx;

	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq;

	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;

	/* List of ready file descriptors */
	struct list_head rdllist;

	/* Lock which protects rdllist and ovflist */
	rwlock_t lock;

	/* RB tree root used to store monitored fd structs */
	struct rb_root_cached rbr;

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 */
	struct epitem *ovflist;

	/* wakeup_source used when ep_scan_ready_list is running */
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;

	struct file *file;

	/* used to optimize loop detection check */
	u64 gen;
	struct hlist_head refs;

#ifdef CONFIG_NET_RX_BUSY_POLL
	/* used to track busy poll napi_id */
	unsigned int napi_id;
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/* tracks wakeup nests for lockdep validation */
	u8 nests;
#endif
};
````

10. 调用``ep_find``函数，从``eventpoll``中的红黑树查找``epitem``(查找动作必须在eventpoll的mtx上锁之后进行)

11. 如果得到的``epitem``已存在则直接返回``-EEXIST``. 否则调用``ep_insert``函数将``epitem``插入到``eventpoll``的红黑树中

12. 检查当前用户支持监听的epoll上限, 可用``cat /proc/sys/fs/epoll/max_user_watches``查看。 如果超出上限则会报错-ENOSPC, 否则会调用``percpu_counter_inc(&ep->user->epoll_watches);``对当前用户的epoll监听数+1(PS.这个函数感觉类似Java中的ThreadLocal，从字面意思上得知是将每个CPU上的缓存监听数+1,也就是说这里改的是CPU缓存)

13. 调用``kmem_cache_zalloc``尝试给``epitem``分配内存. 如果分配失败则会调用``percpu_counter_dec(&ep->user->epoll_watches);``对所有CPU缓存的epoll监听数-1,并报错``-ENOMEM``

14. 调用``ep_rbtree_insert``将epitem插入到红黑树中  

15. 调用``reverse_path_check``检查``epoll``层数，最大4层。其检查的是``epitem->eventpoll->refs(hlist_head)``, 如果检查满足条件则会调用``ep_remove``函数移除对应epoll的epitem，并返回``-EINVAL``

16. 调用``init_poll_funcptr``函数，用队列回调来初始化轮询表(``poll_table``)。 回调函数在``fs/eventpool.c 1239行ep_ptable_queue_proc``

17. 调用``ep_item_poll``函数轮询，轮询epitem

18. 调用``vfs_poll``函数，这里会紧接着调用``sock_poll(net/socket.c)``轮询socket，紧接着到``tcp_poll(net/ipv4/tcp.c)``等待TCP事件

19. 在``tcp_poll``中，会继续调用``sock_poll_wait(include/net/sock.h)``函数对socket轮询并等待，调用``poll_wait(include/linux.poll.h)``, 然后回调poll_table中的_qproc函数，也就是刚才``第16步``初始化好的回调函数``ep_ptable_queue_proc``. 

20. 调用``init_waitqueue_func_entry``初始化等待队列的回调函数，该回调函数位于``fs/eventpoll.c 1134行的ep_poll_callback``,这是等待队列唤醒时的回调函数, 也就是当文件描述符有事件触发时回调

21. 调用``add_wait_queue``函数添加当前``entry``等待队列

22. 检查当前文件是否时"就绪"状态，如果是则将当前epitem添加到就绪队列中

## 6. epoll_wait
1. 调用入口函数``do_epoll_wait(fs.eventpoll.c 2199行)``, 传入event_poll的``fd``、epoll_event指针、最大events长度、超时时间(``ms``). 成功返回时，会返回IO已就绪的fd数目，并且会将对应就绪的fd都赋值给刚才入参的``epoll_event指针``，这是一个数组，直接遍历该数组中的所有fd即可。 返回0时表示超时时间内没有就绪条目，返回-1时表示有报错.

2. 进行基本的参数校验，如最大事件数必须满足 0 < maxevents < EP_MAX_EVENTS, 其中EP_MAX_EVENTS=INT_MAX / sizeof(struct epoll_event), INT_MAX=0x7fffffff即4字节。

3. 校验给定的``epoll_event``指针是否是可写的(当有fd就绪时会赋值到这里，必须检查)，其原理其实就是检查给定的内存地址是否真的处于用户空间

4. 调用``fdget(epfd)``函数，根据之前创建的epoll对象返回的fd，得到对应的``fd``

5. 检查得到的``fd``是否是真的epoll

6. 调用``ep_poll(ep, events, maxevents, timeout)``函数尝试获取就绪事件

7. 调用``ep_events_available(ep)``函数, 传入eventpoll指针, 尝试获取对应eventpoll中的可用事件, 其返回值是可用的就绪事件数. 如果没有可用事件则将当前的等待队列元素的指针添加到eventpoll对应的等待队列, 并调用``schedule_hrtimeout_range(返回0表示超时)``函数，将当前进程sleep直到超时(可中断)

8. 超时或被中断后,会标记当前进程的状态为``TASK_RUNNING``, 并再次尝试获取等待队列的内容.
如果等待队列为空说明是超时导致,则会进入下一次循环; 如果等待队列不为空，则检查是否是超时导致唤醒当前进程, 如果不是超时则调用``eavail = list_empty(&wait.entry)``判断当前等待队列是否为空.

9. 接着``第8点``进入下一次循环后会判断是否超时，如果超时则直接返回0,表示超时时间内无可用的就绪``fd``

10. 如果有可用的就绪事件, 则会调用``ep_send_events``函数尝试将事件转移到用户空间, 即用户代码调用``epoll_wait``时传递的``epoll_event指针``

11. ``ep_send_events``中会调用``ep_wakeup_source(epitem)``获取epitem中的wakeup_source, 如果wakeup_source不为空且状态为active, 则会调用``__pm_stay_awake``唤醒对应的epitem中的wakeup_source

12. 调用``ep_item_poll``尝试轮询事件, vfs_poll ---> sock_poll ---> tcp_poll. tcp_poll中会读取当前socket的情况，如果是TCP_LISTEN状态则调用``inet_csk_listen_poll``函数来轮询监听到的连接数。 如果socket的``connection accept queue``不为空, 则返回``EPOLLIN``

13. 如果``ep_item_poll``的返回值不为0, 则调用函数``epoll_put_uevent``将epoll的事件转移到用户空间中。 当就绪列表中的所有元素都轮询完毕后, 则返回就绪数目。用户代码则可以根据该数值遍历``epoll_event``数组,并读取对应``fd``的数据了
