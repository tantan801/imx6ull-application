# 第三十章 socket 编程基础

> 本章承接第 29 章的理论，正式动手写网络程序。
> 核心就两套 API：**TCP 服务端 6 步**、**TCP 客户端 3 步**。
> 原书定位是"入门"，网络编程本身很深，本章只求建立基本认知。

---

## 30.1 socket 简介

**套接字（socket）** 是 Linux 下的一种进程间通信机制（socket IPC），既能让**不同主机**上的应用程序通信（网络通信），也能让**同一台主机**上的不同程序通信。

通信模式：**客户端 ←→ 服务器**，多个客户端可同时连到同一个服务器。

### socket 到底是什么

内核向应用层提供 socket 接口。socket 是**应用层与 TCP/IP 协议栈之间的中间软件抽象层**，是一组接口。

在设计模式上，**socket 就是门面（Facade）模式**：把复杂的 TCP/IP 协议族藏在接口后面。对用户来说，学会这一组接口就是全部，剩下的事交给 socket 去组织数据、拼成符合协议格式的报文。

**这意味着：我们不需要深入理解 TCP/UDP 细节，只要按 socket 的规矩编程，写出来的程序天然就遵循 TCP/UDP 标准。**

### 为什么 socket 这么重要

因为它是**标准（BSD socket）**：

| 平台 | 是否兼容 BSD socket |
|------|-------------------|
| Linux / Unix | 原生 |
| Windows | 有自己一套 socket 接口（Winsock） |
| LwIP（嵌入式轻量 TCP/IP） | 兼容 BSD Socket |
| RT-Thread（国产 RTOS） | 实现了 BSD socket 标准接口 |

**可移植性**：基于 socket 写的应用，换平台基本不用改代码。这是它成为主流的原因。

---

## 30.2 socket 编程接口介绍

必须先包含两个头文件：

```c
#include <sys/types.h>   /* See NOTES */
#include <sys/socket.h>
```

### 30.2.1 socket() —— 创建套接字

```c
int socket(int domain, int type, int protocol);
```

作用类似 `open()`：**创建一个网络通信端点**。成功返回一个**网络文件描述符**，通常叫 **socket 描述符（socket descriptor）**，后续所有读写都靠它。不需要时用 `close()` 关闭。

失败返回 `-1` 并设置 `errno`。

**参数 ① `domain`：通信域（协议族）**

| 协议族 | 说明 | 帮助 |
|--------|------|------|
| `AF_UNIX` / `AF_LOCAL` | 本地通信 | `unix(7)` |
| **`AF_INET`** | **IPv4 互联网协议** | `ip(7)` |
| `AF_INET6` | IPv6 互联网协议 | `ipv6(7)` |
| `AF_IPX` | IPX - Novell 协议 | |
| `AF_NETLINK` | 内核与用户态接口设备 | `netlink(7)` |
| `AF_X25` | ITU-T X.25 / ISO-8208 | `x25(7)` |
| `AF_AX25` | 业余无线电 AX.25 | |
| `AF_ATMPVC` | 原始 ATM PVC 访问 | |
| `AF_APPLETALK` | AppleTalk | `ddp(7)` |
| `AF_PACKET` | 底层数据包接口 | `packet(7)` |
| `AF_ALG` | 内核加密 API 接口 | |

> **TCP/IP 编程通常选 `AF_INET`**；要支持 IPv6 就选 `AF_INET6`。

**参数 ② `type`：套接字类型**

| 类型 | 说明 |
|------|------|
| **`SOCK_STREAM`** | 有序、可靠、双向、**基于连接**的字节流，保证数据正确送达，**用于 TCP**；支持带外数据传输 |
| **`SOCK_DGRAM`** | 固定长度、**无连接、不可靠**的报文传递，**用于 UDP** |
| `SOCK_SEQPACKET` | 固定长度、有序、可靠、面向连接的报文传递 |
| `SOCK_RAW` | 原始套接字，可访问网络层原始数据包，用得少 |
| `SOCK_RDM` | 可靠数据报层，不保证排序 |
| `SOCK_PACKET` | **已过时，不要在应用中使用** |

**参数 ③ `protocol`：具体协议**

通常填 **`0`**，表示按"通信域 + 套接字类型"选默认协议：

- `AF_INET` + `SOCK_STREAM` → 默认 **TCP**
- `AF_INET` + `SOCK_DGRAM` → 默认 **UDP**

只有当同一组合支持多个协议时，才需要用它挑一个。

**使用示例**

```c
int socket_fd = socket(AF_INET, SOCK_STREAM, 0);   /* 打开套接字 */
if (0 > socket_fd) {
    perror("socket error");
    exit(-1);
}

... ...

close(socket_fd);                                   /* 关闭套接字 */
```

### 30.2.2 bind() —— 绑定地址和端口

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

作用：**把一个 IP 地址 / 端口号绑定到套接字上**。

- 给**客户端**套接字绑地址没什么意义，可以让系统自动选一个默认地址
- 给**服务器**套接字绑一个**众所周知的地址**（固定 IP + 固定端口），因为客户端必须提前知道服务器的地址才能连

成功返回 `0`，失败返回 `-1` 并设置 `errno`。

**`struct sockaddr`：通用地址结构体**

```c
struct sockaddr {
    sa_family_t sa_family;   /* 地址族 */
    char        sa_data[14]; /* 地址数据：IP + 端口都在这里面 */
};
```

问题：`sa_data` 是个 14 字节的 char 数组，把 IP 和端口全塞进去了，**用户根本没法给它赋值**。所以实际编程不用它。

**`struct sockaddr_in`：IPv4 专用地址结构体**

```c
struct sockaddr_in {
    sa_family_t    sin_family;  /* 协议族 */
    in_port_t      sin_port;    /* 端口号 */
    struct in_addr sin_addr;    /* IP 地址 */
    unsigned char  sin_zero[8]; /* 保留未用 */
};
```

**关键点**：`sockaddr_in` 与 `sockaddr` **是并列结构、占用空间一样大**。第一个字段一致，后面就是 `sa_data[14]` 的内容重新切成有名字的成员。所以：

```c
bind(sockfd, (struct sockaddr *)&socket_addr, sizeof(socket_addr));
/*              ↑ 强转一下就能用 */
```

**使用示例**

```c
struct sockaddr_in socket_addr;

memset(&socket_addr, 0x0, sizeof(socket_addr));   /* 一定要清零 */

socket_addr.sin_family = AF_INET;
socket_addr.sin_addr.s_addr = htonl(INADDR_ANY);  /* 监听本机所有 IP（即 0.0.0.0） */
socket_addr.sin_port = htons(5555);               /* 端口号 */

bind(socket_fd, (struct sockaddr *)&socket_addr, sizeof(socket_addr));
```

**`htons()` / `htonl()` 是宏，不是函数**

| 宏 | 作用 | 用在哪 |
|----|------|--------|
| `htons()` | host → network，**short（16 位）** | 端口号 |
| `htonl()` | host → network，**long（32 位）** | IP 地址 |

原因是**大小端问题**：x86/ARM 是小端，网络字节序规定是**大端**，必须转换。
需要包含 `<netinet/in.h>`。

**`INADDR_ANY` 就是 `0.0.0.0`**：含义是"监听本机所有网卡的所有 IP"。如果只写某块网卡的 IP，就只有从那个 IP 进来的连接能被接受。

> **Tips**：`bind()` **不是必须调用**的。只有进程想和某个具体 IP/端口关联时才需要。客户端程序通常不调，让内核自动选地址。

### 30.2.3 listen() —— 进入监听状态

```c
int listen(int sockfd, int backlog);
```

**只能用于服务器进程**，让服务器进入监听状态，等待客户端连接请求。

**调用顺序**：`bind()` 之后、`accept()` 之前。

> ⚠️ **不能在已经连接的套接字上调 `listen()`**——也就是不能对已经 `connect()` 成功的套接字、或 `accept()` 返回的套接字调用它。

**参数 `backlog`：等待连接队列的最大长度**

为什么需要队列？因为 TCP 建连接是个过程，同时尝试连接的客户端可能很多，服务器来不及逐个处理。内核会在自己的进程空间维护一个队列，把这些**尚未处理的连接请求**按先来后到排队。

`backlog` 就是这个队列的上限。**队列满时**，新到的客户端连接请求可能收到"连接失败"错误，本次请求被丢弃不处理。

> 实践中常写 5、10、50 这类值。写太大内核会截断到 `net.core.somaxconn`。

### 30.2.4 accept() —— 取出一个连接

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

服务器调用 `listen()` 后进入监听状态，用 `accept()` **获取客户端的连接请求并建立连接**。

**服务器标准流程（4 步）**

1. `socket()` 打开套接字
2. `bind()` 绑定 IP + 端口
3. `listen()` 进入监听状态
4. `accept()` 处理到来的连接请求

**阻塞行为**：如果没有客户端连进来（等待队列也是空的），`accept()` 会**阻塞**，直到有请求到达。

**最关键的一点：`accept()` 会返回一个"新的套接字"。**

| 套接字 | 来源 | 用途 |
|--------|------|------|
| **监听套接字** `sockfd` | `socket()` | 只负责接客（`listen`/`accept`），**不用于传数据** |
| **连接套接字** `connfd` | `accept()` | 代表"服务器与某个客户端的一条连接"，**收发数据用它** |

理解 `accept()` 的关键就是这个：**它新建一个套接字，专门对应那个调了 `connect()` 的客户端**。一个服务器可以 accept 出很多个 connfd，每个对应一个客户端。

**参数**

- `addr`：**传出参数**，返回已连接客户端的 IP 地址和端口号
- `addrlen`：传入 `addr` 指向对象的字节长度
- **不关心客户端信息时，两个都传 `NULL`**

出错返回 `-1` 并设置 `errno`。

### 30.2.5 connect() —— 客户端发起连接

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**只用于客户端**。把 `sockfd` 连接到远程服务器，`addr` 指定服务器的 IP + 端口。

**TCP 和 UDP 的差别（很重要）**

| 协议 | 调 connect() 实际发生什么 |
|------|-------------------------|
| **TCP** | 触发**三次握手**，最终建立一条真正的 TCP 连接 |
| **UDP** | **不发送任何数据**，只是在 `sockfd` 里记录下服务器的 IP 和端口（之后 send 就默认发给它） |

成功返回 `0`，失败 `-1` 并设置 `errno`。

### 30.2.6 发送和接收

连接建立后，用套接字描述符收发数据。用法和读写普通文件几乎一样：

- **收**：`read()` 或 `recv()`
- **发**：`write()` 或 `send()`

> **用哪个描述符？** 客户端用 `socket()` 返回的；**服务器必须用 `accept()` 返回的那个 `connfd`**，不是监听套接字。

**`read()`**

熟悉的文件读函数。套接字描述符本身就是文件描述符，所以 `read()` 的 `fd` 参数直接传套接字描述符即可。

返回值可能小于请求字节数（接近结尾、从管道/终端读、被信号中断等），**这不算错误**。返回 `0` 表示已到末尾（对 socket 来说通常是对端关闭了连接）。

**`recv()`**

```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

功能和 `read()` 类似，区别是**能通过 `flags` 控制接收行为**。

| 标志 | 描述 |
|------|------|
| `MSG_CMSG_CLOEXEC` | 为 UNIX 域套接字上接收的文件描述符设置执行时关闭标志 |
| `MSG_DONTWAIT` | **启动非阻塞操作**（相当于 `O_NONBLOCK`） |
| `MSG_ERRQUEUE` | 接收错误信息作为辅助数据 |
| `MSG_OOB` | 如果协议支持，获取**带外数据** |
| `MSG_PEEK` | **返回数据包内容但不真正取走**，下次 read/recv 还能拿到 |
| `MSG_TRUNC` | 即使数据包被截断，也返回数据包的真实长度 |
| `MSG_WAITALL` | **等到所有请求数据都到齐才返回**（仅 `SOCK_STREAM`） |

**注意 `MSG_WAITALL`**：对 `SOCK_STREAM` 可以阻止"少读"，但对 `SOCK_DGRAM` / `SOCK_SEQPACKET` **没影响**——因为基于报文的套接字一次读取本来就返回整个报文。

**通常 `flags` 填 `0`。**

返回 `0` 的场景：发送者已调 `shutdown` 结束传输、或协议支持按序关闭且发送端已关闭，且所有数据都收完了。

成功返回实际读到的字节数。

**`write()`**

向套接字写入数据。成功返回写入字节数，失败返回 `-1` 并设置 `errno`。

**`send()`**

```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

和 `write()` 类似，区别同样在 `flags`：

| 标志 | 描述 |
|------|------|
| `MSG_CONFIRM` | 提供链路层反馈以保持地址映射有效 |
| `MSG_DONTROUTE` | 不要把数据包路由出本地网络 |
| `MSG_DONTWAIT` | 允许非阻塞操作（等价 `O_NONBLOCK`） |
| `MSG_EOR` | 如果协议支持，标志记录结束 |
| `MSG_MORE` | 延迟发送数据包，允许继续写更多数据 |
| `MSG_NOSIGNAL` | **写已断开的连接时不产生 SIGPIPE 信号**（很实用，能避免进程被信号杀掉） |
| `MSG_OOB` | 如果协议支持，发送带外数据 |

> ⚠️ **`send()` 成功返回 ≠ 对端进程收到了数据。** 能保证的只是：数据已经**无错误地交给了网络驱动**。

### 30.2.7 close() —— 关闭套接字

不再需要套接字描述符时，调用 `close()` 关闭它并释放资源。

> 注意：服务器要关**两个**——`connfd`（连接）和 `sockfd`（监听）。

---

## 30.3 IP 地址格式转换函数

**问题**：人看的是点分十进制字符串（`192.168.1.110`），机器要的是二进制地址（`struct in_addr`），两者之间必须转换。

**5 个函数**：`inet_aton`、`inet_addr`、`inet_ntoa`、`inet_ntop`、`inet_pton`

需要包含：

```c
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
```

### 30.3.1 已废弃的三个：inet_aton / inet_addr / inet_ntoa

在点分十进制和二进制之间转换，但**已被废弃**，只支持 IPv4。老代码里能见到，**新代码不要用**。

### 30.3.2 inet_pton() —— 字符串 → 二进制

```c
int inet_pton(int af, const char *src, void *dst);
```

`pton` = **p**resentation **to** **n**umber（点分字符串 → 网络二进制）。

| 参数 | 说明 |
|------|------|
| `af` | `AF_INET`（IPv4）或 `AF_INET6`（IPv6） |
| `src` | 点分十进制字符串 |
| `dst` | `AF_INET` 时指向 `struct in_addr`；`AF_INET6` 时指向 `struct in6_addr` |

**返回值**

| 返回 | 含义 |
|------|------|
| `1` | 转换成功 |
| `0` | `src` 不是该地址族的有效网络地址字符串 |
| `-1` | `af` 不是有效地址族，并置 `errno = EAFNOSUPPORT` |

**使用示例**

```c
#include <stdio.h>
#include <stdlib.h>
#include <arpa/inet.h>

#define IPV4_ADDR "192.168.1.222"

int main(void)
{
    struct in_addr addr;

    inet_pton(AF_INET, IPV4_ADDR, &addr);
    printf("ip addr: 0x%x\n", addr.s_addr);
    exit(0);
}
```

![30-1 图30.3.1 测试结果.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-1%20%E5%9B%BE30.3.1%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.jpg)

> 输出是 `0xde01a8c0`，看着像乱码——这正说明**内存里存的是大端序**：`192.168.1.222` → 字节 `C0 A8 01 DE`，按小端机器读成 32 位数就是 `0xde01a8c0`。

### 30.3.3 inet_ntop() —— 二进制 → 字符串

```c
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

`ntop` = **n**umber **to** **p**resentation（网络二进制 → 点分字符串），与 `inet_pton()` 相反。

- `src`：指向 `struct in_addr` 或 `struct in6_addr`
- `dst`：输出缓冲区
- `size`：缓冲区大小

**返回值**：成功返回 `dst` 指针；`size` 太小则返回 `NULL` 并置 `errno = ENOSPC`。

**使用示例**

```c
#include <stdio.h>
#include <stdlib.h>
#include <arpa/inet.h>

int main(void)
{
    struct in_addr addr;
    char buf[20] = {0};

    addr.s_addr = 0xde01a8c0;
    inet_ntop(AF_INET, &addr, buf, sizeof(buf));
    printf("ip addr: %s\n", buf);
    exit(0);
}
```

![30-2 图30.3.2 测试结果.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-2%20%E5%9B%BE30.3.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.jpg)

输出 `192.168.1.222`，和前面的输入对上了。

### 30.3.4 一对函数怎么选

| 场景 | 用哪个 |
|------|--------|
| 代码里写死 IP（`#define SERVER_IP "192.168.1.150"`） | `inet_pton()` 转成 `struct in_addr` |
| 打印客户端 IP | `inet_ntop()` 转成字符串 |
| 只需要 IPv4、老代码 | `inet_aton`/`inet_ntoa`（能跑但别用） |

---

## 30.4 socket 编程实战

实现一个简单服务器 + 一个简单客户端：客户端连上服务器后，把用户输入的字符串发过去，服务器收下来打印。

### 30.4.1 服务器程序（6 步）

1. `socket()` 打开套接字，得到套接字描述符
2. `bind()` 绑定 IP 地址和端口号
3. `listen()` 进入监听状态
4. `accept()` 获取客户端连接并建立连接
5. `read`/`recv`、`write`/`send` 与客户端通信
6. `close()` 关闭套接字

```c
/***************************************************************
 文件名 : socket_server.c
 描述 : 一个简单地 TCP 服务器应用程序示例代码
 ***************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>

#define SERVER_PORT 8888    /* 端口不能冲突，不常用的端口号通常大于 5000 */

int main(void)
{
    struct sockaddr_in server_addr = {0};
    struct sockaddr_in client_addr = {0};
    char ip_str[20] = {0};
    int sockfd, connfd;
    int addrlen = sizeof(client_addr);
    char recvbuf[512];
    int ret;

    /* ① 打开套接字，得到套接字描述符 */
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (0 > sockfd) {
        perror("socket error");
        exit(EXIT_FAILURE);
    }

    /* ② 将套接字与指定端口号进行绑定 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);   /* 监听本机所有 IP */
    server_addr.sin_port = htons(SERVER_PORT);

    ret = bind(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    if (0 > ret) {
        perror("bind error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    /* ③ 使服务器进入监听状态 */
    ret = listen(sockfd, 50);
    if (0 > ret) {
        perror("listen error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    /* ④ 阻塞等待客户端连接 */
    connfd = accept(sockfd, (struct sockaddr *)&client_addr, &addrlen);
    if (0 > connfd) {
        perror("accept error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    printf("有客户端接入...\n");
    inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, ip_str, sizeof(ip_str));
    printf("客户端主机的 IP 地址: %s\n", ip_str);
    printf("客户端进程的端口号: %d\n", client_addr.sin_port);

    /* ⑤ 接收客户端发送过来的数据 */
    for ( ; ; ) {
        memset(recvbuf, 0x0, sizeof(recvbuf));          /* 缓冲区清零 */

        ret = recv(connfd, recvbuf, sizeof(recvbuf), 0);
        if (0 >= ret) {
            perror("recv error");
            close(connfd);
            break;
        }

        printf("from client: %s\n", recvbuf);

        /* 收到 "exit" 就退出 */
        if (0 == strncmp("exit", recvbuf, 4)) {
            printf("server exit...\n");
            close(connfd);
            break;
        }
    }

    /* ⑥ 关闭套接字 */
    close(sockfd);
    exit(EXIT_SUCCESS);
}
```

**几点说明**

- `SERVER_PORT` 设成 **8888**。端口不能和别的服务冲突，**自定义服务一般选 5000 以上**。
- `sin_addr.s_addr = htonl(INADDR_ANY)` = 监听 `0.0.0.0`，任何网卡来的连接都收。
- 这个服务器**只能处理一个客户端**，accept 一次就进收发循环了。要多客户端就得上 `fork` / 线程 / `select`·`poll` / `epoll`。

### 30.4.2 客户端程序（3 步）

1. `socket()` 打开套接字
2. `connect()` 连接服务器
3. `send()` 发数据（可循环）

```c
/***************************************************************
 文件名 : socket_client.c
 描述 : 一个简单地 TCP 客户端应用程序示例代码
 ***************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>

#define SERVER_PORT 8888              /* 服务器的端口号 */
#define SERVER_IP   "192.168.1.150"   /* 服务器的 IP 地址 */

int main(void)
{
    struct sockaddr_in server_addr = {0};
    char buf[512];
    int sockfd;
    int ret;

    /* 打开套接字，得到套接字描述符 */
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (0 > sockfd) {
        perror("socket error");
        exit(EXIT_FAILURE);
    }

    /* 调用 connect 连接远端服务器 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);                     /* 端口号 */
    inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr);          /* IP 地址 */

    ret = connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    if (0 > ret) {
        perror("connect error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    printf("服务器连接成功...\n\n");

    /* 向服务器发送数据 */
    for ( ; ; ) {
        memset(buf, 0x0, sizeof(buf));                  /* 清理缓冲区 */

        printf("Please enter a string: ");
        fgets(buf, sizeof(buf), stdin);

        ret = send(sockfd, buf, strlen(buf), 0);
        if (0 > ret) {
            perror("send error");
            break;
        }

        if (0 == strncmp(buf, "exit", 4))               /* 输入 exit 退出 */
            break;
    }

    close(sockfd);
    exit(EXIT_SUCCESS);
}
```

**注意**：`SERVER_IP` 要改成**你服务器实际的 IP**。服务器端端口是 8888，客户端必须一致。

### 30.4.3 编译

socket 属于 libc，**不需要 `-l` 链接额外库**：

```bash
. /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi

# 交叉编译（给板子）
${CC} -o server socket_server.c
${CC} -o client socket_client.c

# 如果在 Ubuntu 本机跑客户端，用本机 gcc
gcc -o client_pc socket_client.c
```

![30-3 图30.4.1 编译客户端应用程序和服务器应用程序 编译得到 c.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-3%20%E5%9B%BE30.4.1%20%E7%BC%96%E8%AF%91%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E5%92%8C%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%20%E7%BC%96%E8%AF%91%E5%BE%97%E5%88%B0%20c.jpg)

### 30.4.4 测试

原书方案：**服务器跑在开发板，客户端跑在 Ubuntu**。（两端都跑在板子或都跑在 Ubuntu 也行）

**步骤 1**：把 server 拷到板子 `/home/root`

```bash
scp server root@192.168.77.148:/home/root/
```

![30-4 图30.4.2 将服务器执行文件拷贝到开发板 在开发板执行 se.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-4%20%E5%9B%BE30.4.2%20%E5%B0%86%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%E6%8B%B7%E8%B4%9D%E5%88%B0%E5%BC%80%E5%8F%91%E6%9D%BF%20%E5%9C%A8%E5%BC%80%E5%8F%91%E6%9D%BF%E6%89%A7%E8%A1%8C%20se.jpg)

**步骤 2**：板上执行 server（会阻塞在 accept）

![30-5 图30.4.3 先执行服务器应用程序 接着在 Ubuntu 系统.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-5%20%E5%9B%BE30.4.3%20%E5%85%88%E6%89%A7%E8%A1%8C%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%20%E6%8E%A5%E7%9D%80%E5%9C%A8%20Ubuntu%20%E7%B3%BB%E7%BB%9F.png)

```bash
cd /home/root
./server
```

**步骤 3**：Ubuntu 上执行客户端

![30-6 图30.4.4 执行客户端应用程序 客户端运行之后将会去连接远端.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-6%20%E5%9B%BE30.4.4%20%E6%89%A7%E8%A1%8C%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%20%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%90%E8%A1%8C%E4%B9%8B%E5%90%8E%E5%B0%86%E4%BC%9A%E5%8E%BB%E8%BF%9E%E6%8E%A5%E8%BF%9C%E7%AB%AF.png)

```bash
./client_pc
# 打印“服务器连接成功...”
```

**步骤 4**：客户端连上后，服务器也会打印接入信息

![30-7 图30.4.5 服务器监测到客户端连接.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-7%20%E5%9B%BE30.4.5%20%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%9B%91%E6%B5%8B%E5%88%B0%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%9E%E6%8E%A5.png)

服务器会打印：

```
有客户端接入...
客户端主机的 IP 地址: 192.168.1.xxx
客户端进程的端口号: 54321
```

> 注意 `client_addr.sin_port` **没有用 `ntohs()` 转换**，所以打印出来是**字节序反了的值**（原书代码就是这样）。想打印正确端口号应写 `ntohs(client_addr.sin_port)`。

**步骤 5**：客户端输入字符串发给服务器

![30-8 图30.4.6 输入字符串信息.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-8%20%E5%9B%BE30.4.6%20%E8%BE%93%E5%85%A5%E5%AD%97%E7%AC%A6%E4%B8%B2%E4%BF%A1%E6%81%AF.jpg)

![30-9 图30.4.7 服务器接收到客户端发送的信息.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E7%AB%A0%20socket%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/30-9%20%E5%9B%BE30.4.7%20%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A5%E6%94%B6%E5%88%B0%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%8F%91%E9%80%81%E7%9A%84%E4%BF%A1%E6%81%AF.jpg)

服务器收到并打印 `from client: xxxx`。输入 `exit` 双方退出。

---

## 常见坑（实际调试最容易踩的）

| 现象 | 原因 | 解决 |
|------|------|------|
| `bind error: Address already in use` | 上次进程没退干净，端口还被占（TIME_WAIT） | `setsockopt` 加 `SO_REUSEADDR`；或换端口；或 `netstat -anp \| grep 8888` 找占用 |
| `connect error: Connection refused` | 服务器没启动 / 端口不对 / 防火墙 | 先起服务器，核对端口 |
| `connect error: No route to host` | 两端不在同一网段，或板子网线没通 | 先 `ping` 通 |
| 客户端连上了但收不到回包 | 服务器用了 `sockfd` 而不是 `connfd` 收发 | 收发必须用 `accept()` 返回的 fd |
| 服务器只能接一个客户端 | `accept()` 只调了一次 | 循环 accept，或 fork/线程/epoll |
| `bind error: Permission denied` | 绑了 1024 以下的端口且非 root | 换 5000 以上端口 |
| 报文被"粘"在一起 | TCP 是**字节流**，没有消息边界 | 自定义长度头 / 分隔符，或改用 UDP |

---

## 本章小结

| 函数 | 谁用 | 一句话 |
|------|------|--------|
| `socket()` | 双方 | 建套接字，返回 fd（类似 open） |
| `bind()` | 服务器（客户端可选） | 绑 IP + 端口，`INADDR_ANY` = 0.0.0.0 |
| `listen()` | 仅服务器 | 进入监听，`backlog` = 等待队列长度 |
| `accept()` | 仅服务器 | **阻塞**等连接，**返回新的 connfd** |
| `connect()` | 仅客户端 | TCP 触发三次握手；UDP 只记地址 |
| `recv()` / `read()` | 双方 | 收数据，`recv` 能用 flags |
| `send()` / `write()` | 双方 | 发数据，成功 ≠ 对方已收到 |
| `close()` | 双方 | 关套接字（服务器要关两个） |
| `inet_pton()` | 双方 | 字符串 → 二进制（写死 IP 时用） |
| `inet_ntop()` | 双方 | 二进制 → 字符串（打印 IP 时用） |

**两条流程背下来就够入门：**

```
服务器：socket → bind → listen → accept → recv/send 循环 → close
客户端：socket → connect → send/recv 循环 → close
```

> 原书的定位很明确：本章**只求入门**。网络编程是很专业的方向，真要深入得看《UNIX 网络编程》那类专著。
