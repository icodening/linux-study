# epoll方式
## 主体流程
> 1. socket
> 2. bind
> 3. listen
> 4. epoll_create
> 5. epoll_ctl
> 6. epoll_wait
  

## 步骤详解
### 1. socket
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

### 2. bind
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

### 3. listen
``TODO``