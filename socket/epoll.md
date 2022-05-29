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

4. 继续调用__sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0)，其中第一个参数是获取当前进程的namespace proxy，再根据namespace proxy获取其网络命名空间
  
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