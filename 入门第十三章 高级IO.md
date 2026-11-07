# 第十三章 高级 I/O

> 第二章到第四章讲的是"能用"的文件 I/O，本章讲"好用"的那部分：面对管道、设备、socket 这类会阻塞的文件，怎么不被卡死、怎么同时盯住多个文件、怎么让内核主动通知、怎么绕开 read/write 直接操作内存、怎么让多个进程不打架。
>
> 本章主题：阻塞/非阻塞 I/O；I/O 多路复用（`select`、`poll`）；异步 I/O（`SIGIO`、实时信号 + `sigaction`）；存储映射 I/O（`mmap`/`munmap`/`mprotect`/`msync`）；文件锁（`flock`/`fcntl`/`lockf`）。

---

## 13.1 非阻塞 I/O

"阻塞"就是**进入休眠、交出 CPU**，`wait()`、`pause()`、`sleep()` 都会阻塞。I/O 的阻塞指的是读写操作：

- **阻塞式**：数据没准备好时读操作让调用者阻塞，有数据才唤醒；
- **非阻塞式**：没数据也不阻塞，**立即返回错误**。

一个关键事实：**普通文件的读写永远不会阻塞**，不管读写多少字节，`read()`/`write()` 一定在有限时间内返回——普通文件天生是非阻塞的。会阻塞的是管道、网络设备、字符设备文件，它们既可阻塞也可非阻塞。

### 13.1.1 阻塞 I/O 与非阻塞 I/O 读文件

切换只有一个开关：`open()` 时给 `flags` 加 `O_NONBLOCK`，之后所有 I/O 都是非阻塞；不加则默认阻塞。对普通文件加不加没区别。

原书用读鼠标做对比。输入设备的设备文件在 `/dev/input/` 下，通常是 `mouseX`，也可能是 `eventX`，用 `od` 试读确认：

```bash
sudo od -x /dev/input/event3
```

> 需要 `sudo`，Ubuntu 下普通用户不能读写设备文件。执行后移动/按下/松开鼠标，终端有数据打印就说明找对了设备。原书环境是 `/dev/input/event3`。

![图13.1.1 输入设备对应的设备文件](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-1%20%E5%9B%BE13.1.1%20%E8%BE%93%E5%85%A5%E8%AE%BE%E5%A4%87%E5%AF%B9%E5%BA%94%E7%9A%84%E8%AE%BE%E5%A4%87%E6%96%87%E4%BB%B6%20%E9%80%9A%E5%B8%B8%E6%83%85%E5%86%B5%E4%B8%8B%E6%98%AF%20mouse.png)

![图13.1.2 读取鼠标打印信息](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-2%20%E5%9B%BE13.1.2%20%E8%AF%BB%E5%8F%96%E9%BC%A0%E6%A0%87%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF.png)

**示例代码 13.1.1：阻塞式读取**（只读一次）

```c
char buf[100];
int fd, ret;

fd = open("/dev/input/event3", O_RDONLY);      /* 没有 O_NONBLOCK */
if (-1 == fd) { perror("open error"); exit(-1); }

memset(buf, 0, sizeof(buf));
ret = read(fd, buf, sizeof(buf));
if (0 > ret) { perror("read error"); close(fd); exit(-1); }

printf("成功读取<%d>个字节数据\n", ret);
close(fd);
```

运行时程序不结束、一直占着终端也没输出，因为 `read()` 阻塞了；移动或按下鼠标后阻塞解除、`read` 返回。

![图13.1.3 阻塞](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-3%20%E5%9B%BE13.1.3%20%E9%98%BB%E5%A1%9E%20%E6%89%A7%E8%A1%8C%E7%A8%8B%E5%BA%8F%E4%B9%8B%E5%90%8E%E5%8F%91%E7%8E%B0%E7%A8%8B%E5%BA%8F%E6%B2%A1%E6%9C%89%E7%AB%8B%E5%8D%B3%E7%BB%93%E6%9D%9F%E8%80%8C%E6%98%AF%E4%B8%80%E7%9B%B4%E5%8D%A0.png)

![图13.1.4 移动鼠标](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-4%20%E5%9B%BE13.1.4%20%E7%A7%BB%E5%8A%A8%E9%BC%A0%E6%A0%87%20%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF%E6%8F%90%E7%A4%BA%E6%AD%A4%E6%AC%A1%20read%20%E6%88%90%E5%8A%9F%E8%AF%BB%E5%8F%96%E4%BA%86.png)

注意打印的是读了 **48** 字节，而程序请求 100 字节。原书说明这属于输入子系统的数据结构问题，第二篇再讲。

**示例代码 13.1.2：非阻塞式**——只改 `open`：

```c
fd = open("/dev/input/event3", O_RDONLY | O_NONBLOCK);
```

程序立刻结束并 `perror` 打印 `Resource temporarily unavailable`（资源暂时不可用），因为鼠标没动、无数据可读，非阻塞方式直接失败。

![图13.1.5 非阻塞](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-5%20%E5%9B%BE13.1.5%20%E9%9D%9E%E9%98%BB%E5%A1%9E%20%E6%89%A7%E8%A1%8C%E7%A8%8B%E5%BA%8F%E4%B9%8B%E5%90%8E%E7%A8%8B%E5%BA%8F%E7%AB%8B%E9%A9%AC%E5%B0%B1%E7%BB%93%E6%9D%9F%E4%BA%86%E5%B9%B6%E4%B8%94%E8%B0%83%E7%94%A8%20r.png)

**示例代码 13.1.3：轮询 + 非阻塞**——死循环一直试，直到读到数据：

```c
memset(buf, 0, sizeof(buf));
for ( ; ; ) {
    ret = read(fd, buf, sizeof(buf));
    if (0 < ret) {
        printf("成功读取<%d>个字节数据\n", ret);
        close(fd);
        exit(0);
    }
}
```

### 13.1.2 阻塞 I/O 的优点与缺点

| | 阻塞 I/O | 非阻塞 I/O |
|---|---|---|
| 无数据时 | 进程挂起、休眠阻塞，交出 CPU | 立即返回，不挂起 |
| CPU 占用 | 几乎为 0 | 轮询时几乎 100% |
| 后续行为 | 有数据自动唤醒 | 一直轮询等，或直接放弃 |
| 优点 | 提升 CPU 处理效率，把 CPU 让给别人 | 不会卡在某一个文件上 |
| 缺点 | 一卡住就做不了别的事 | 死循环轮询极度浪费 CPU |

`top` 看示例 13.1.3，CPU 占用几乎 100%——一个进程吃满 CPU 在系统里非常危险。

![图13.1.6 CPU 占用率](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-6%20%E5%9B%BE13.1.6%20CPU%20%E5%8D%A0%E7%94%A8%E7%8E%87.png)

所以单看"读一个鼠标"，阻塞式完胜。为什么还要学非阻塞？下一小节给出阻塞式的困境。

### 13.1.3 使用非阻塞 I/O 实现并发读取

需求：同时读鼠标和键盘。键盘是标准输入设备，进程从父进程继承了标准输入/输出/错误，键盘对应 fd **0**，直接用，不必 `open`。

**示例代码 13.1.4：阻塞式同时读取**（先鼠标后键盘）

```c
fd = open(MOUSE, O_RDONLY);               /* MOUSE 即 "/dev/input/event3" */

ret = read(fd, buf, sizeof(buf));         /* 没数据就一直阻塞在这里 */
printf("鼠标: 成功读取<%d>个字节数据\n", ret);

ret = read(0, buf, sizeof(buf));          /* 鼠标不来数据，根本走不到这里 */
printf("键盘: 成功读取<%d>个字节数据\n", ret);
```

实测必须先动鼠标、再按键盘（还要按回车）才能走完——**这就是阻塞 I/O 的困境：无法并发读取**。绕开可以用多线程（一个读鼠标、一个读键盘）或多进程（父读鼠标、子读键盘），但本章走非阻塞路线。

鼠标 fd 是 `open` 来的，加 `O_NONBLOCK` 即可；键盘 fd 0 是继承来的，得用 `fcntl()` 改 flags：

```c
int flag;

flag = fcntl(0, F_GETFL);      /* 取原来的 flag */
flag |= O_NONBLOCK;            /* 加上 O_NONBLOCK */
fcntl(0, F_SETFL, flag);       /* 写回 */
```

**示例代码 13.1.5：非阻塞式同时读取**

```c
fd = open(MOUSE, O_RDONLY | O_NONBLOCK);

flag = fcntl(0, F_GETFL);
flag |= O_NONBLOCK;
fcntl(0, F_SETFL, flag);

for ( ; ; ) {
    ret = read(fd, buf, sizeof(buf));      /* 读鼠标 */
    if (0 < ret) printf("鼠标: 成功读取<%d>个字节数据\n", ret);

    ret = read(0, buf, sizeof(buf));       /* 读键盘 */
    if (0 < ret) printf("键盘: 成功读取<%d>个字节数据\n", ret);
}
```

先动鼠标还是先按键盘都能读到，问题解决。

![图13.1.7 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-7%20%E5%9B%BE13.1.7%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

代价是 **CPU 占用率特别高**，对整个系统有副作用。这就轮到 I/O 多路复用登场。

---

## 13.2 I/O 多路复用

### 13.2.1 何为 I/O 多路复用

I/O 多路复用（IO multiplexing）通过一种机制**监视多个文件描述符**，一旦其中某个可以执行 I/O，就通知应用程序做相应读写。它要解决的是：并发式 I/O 场景中进程阻塞在某个特定 I/O 系统调用上的问题，让进程不阻塞于某一个文件。

所以它常用于并发式的非阻塞 I/O（多路非阻塞）。两个功能几乎相同的系统调用：`select()` 和 `poll()`，细节上有些差别。

特征是：**外部阻塞式，内部监视多路 I/O**——进程还是阻塞的，但阻塞在一个"监视者"身上，而不是某个具体文件上。

### 13.2.2 select() 函数

```c
#include <sys/select.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

调用后会一直阻塞，直到一个或多个 fd 成为就绪态（可读或可写）。

| 参数 | 含义 |
|---|---|
| `nfds` | 三个集合中**最大 fd 编号 + 1** |
| `readfds` | 检测"读是否就绪"的 fd 集合 |
| `writefds` | 检测"写是否就绪"的 fd 集合 |
| `exceptfds` | 检测异常情况是否发生的 fd 集合 |
| `timeout` | 阻塞时间上限；`NULL` 表示一直阻塞 |

`fd_set` 是以位掩码实现的 fd 集合，不用关心内部结构——所有操作都用 4 个宏完成。

对某个集合不感兴趣就传 `NULL`；三个都传 `NULL` 时 `select()` 退化成类似 `sleep()` 的休眠函数，用 `timeout` 控制睡多久。

> **Tips**：`exceptfds` 的"异常"并不是 fd 上出现了错误。

`timeout` 三种用法：

- 传 `NULL`：一直阻塞，直到有 fd 就绪或被信号打断；
- 指向的 `struct timeval` 两个成员都为 0：不阻塞，只轮询一遍集合，立即返回；
- 其它值：阻塞上限，期间有 fd 就绪就提前返回，超时也返回。

`select()` 在以下任一情况解除阻塞：

⚫ `readfds`/`writefds`/`exceptfds` 中至少有一个 fd 成为就绪态；
⚫ 该调用被信号处理函数中断；
⚫ `timeout` 指定的时间上限已超时。

### 13.2.3 FD_CLR()、FD_ISSET()、FD_SET()、FD_ZERO()

```c
#include <sys/select.h>

void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```

⚫ `FD_ZERO()`：把集合清空；
⚫ `FD_SET()`：把 fd 加入集合；
⚫ `FD_CLR()`：把 fd 移出集合；
⚫ `FD_ISSET()`：fd 在集合中返回 true，否则 false。

集合容量上限由常量 **`FD_SETSIZE`** 决定，Linux 下为 **1024**。定义集合后**必须先用 `FD_ZERO()` 初始化**，再添加关心的 fd：

```c
fd_set fset;

FD_ZERO(&fset);      /* 初始化为空 */
FD_SET(3, &fset);
FD_SET(4, &fset);
FD_SET(5, &fset);
```

⚠️ **关键坑**：`select()` 内部会**修改**这三个集合，返回时集合里只剩"已就绪"的 fd。例如调用前 `readfds` 里有 3、4、5，返回时只有 4 就绪，集合里就只剩 4。**所以在循环里重复调用 `select()`，每轮都必须重新初始化并重新添加**。

### 13.2.4 select() 的返回值

| 返回值 | 含义 |
|---|---|
| `-1` | 出错并设置 `errno`：`EBADF`（集合中有非法 fd）、`EINTR`（被信号处理函数中断）、`EINVAL`、`ENOMEM` 等 |
| `0` | 超时，没有任何 fd 就绪；此时三个集合都会被**清空** |
| `>0` | 有 fd 就绪；值是**就绪 fd 的个数** |

返回正数时，每个集合都要用 `FD_ISSET()` 逐个检查才知道发生了什么事件。同一个 fd 若在多个集合中指定、且多个事件都就绪，会被**重复计数**——返回值是三个集合中标记为就绪的 fd 总数。

### 13.2.5 使用示例

示例代码 13.2.1：用 `select()` 同时读键盘和鼠标，读 5 次。因为反复调用 `select()`，每轮都要重置 `rdfds`。

```c
fd_set rdfds;

fd = open(MOUSE, O_RDONLY | O_NONBLOCK);

flag = fcntl(0, F_GETFL);            /* 键盘设为非阻塞 */
flag |= O_NONBLOCK;
fcntl(0, F_SETFL, flag);

while (loops--) {
    FD_ZERO(&rdfds);                 /* 每轮都要重新初始化！ */
    FD_SET(0, &rdfds);               /* 加键盘 */
    FD_SET(fd, &rdfds);              /* 加鼠标 */

    ret = select(fd + 1, &rdfds, NULL, NULL, NULL);
    if (0 > ret) { perror("select error"); goto out; }
    else if (0 == ret) { fprintf(stderr, "select timeout.\n"); continue; }

    if (FD_ISSET(0, &rdfds)) {       /* 键盘就绪 */
        ret = read(0, buf, sizeof(buf));
        if (0 < ret) printf("键盘: 成功读取<%d>个字节数据\n", ret);
    }

    if (FD_ISSET(fd, &rdfds)) {      /* 鼠标就绪 */
        ret = read(fd, buf, sizeof(buf));
        if (0 < ret) printf("鼠标: 成功读取<%d>个字节数据\n", ret);
    }
}
```

`timeout` 传 `NULL`、只关心可读，所以 `writefds`/`exceptfds` 都传 `NULL`；`nfds` 传 `fd + 1`（鼠标 fd 比 0 大）。两个 fd 都没数据时 `select()` 阻塞，一有输入事件即返回。

![图13.2.1 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-8%20%E5%9B%BE13.2.1%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81%2013.2.1%20%E5%B0%86%E9%BC%A0%E6%A0%87%E5%92%8C%E9%94%AE%E7%9B%98%E9%83%BD.png)

示例里两个 fd 都设了非阻塞，其实设成阻塞也没问题——`select()` 返回时数据已可读，两种方式读都不会阻塞。

### 13.2.6 poll() 函数

`poll()` 与 `select()` 很相似，接口不同：`select()` 用三个 `fd_set`，`poll()` 用**一个 `struct pollfd` 数组**，每个元素指定一个 fd 和关心的条件。

```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

| 参数 | 含义 |
|---|---|
| `fds` | 指向 `struct pollfd` 数组 |
| `nfds` | 数组元素个数（`nfds_t` 实际是无符号整型） |
| `timeout` | 阻塞行为，单位**毫秒** |

| `timeout` 值 | 行为 |
|---|---|
| `-1` | 一直阻塞（等同 `select()` 的 `NULL`），直到有 fd 就绪或捕获到信号 |
| `0` | 不阻塞，只检查一次哪个 fd 就绪 |
| `>0` | 最多阻塞 `timeout` 毫秒 |

### 13.2.7 struct pollfd 结构体

```c
struct pollfd {
    int   fd;        /* file descriptor */
    short events;    /* requested events */
    short revents;   /* returned events */
};
```

`fd` 是文件描述符；`events` 和 `revents` 都是位掩码。调用者初始化 `events` 告诉内核要检查什么；`poll()` 返回时由内核设置 `revents` 说明实际发生了哪些事件（**`poll()` 不会改 `events`**）。`events` 可用位或组合多个标志。表 13.2.1：

| 组 | 标志 | 说明 |
|---|---|---|
| 可读 | `POLLIN` | 有数据可以读取 |
| | `POLLRDNORM` | 相等于 `POLLIN` |
| | `POLLRDBAND` | 可以读取优先级数据（Linux 上通常不使用） |
| | `POLLPRI` | 可读取高优先级数据 |
| | `POLLRDHUP` | 对端套接字关闭 |
| 可写 | `POLLOUT` | 可写入数据 |
| | `POLLWRNORM` | 相等于 `POLLOUT` |
| | `POLLWRBAND` | 优先级数据可写入 |
| 附加信息 | `POLLERR` | 有错误发生 |
| | `POLLHUP` | 出现挂断 |
| | `POLLNVAL` | 文件描述符未打开 |
| | `POLLMSG` | Linux 中不使用 |

第三组（`POLLERR`、`POLLHUP`、`POLLNVAL`）是**只能从 `revents` 得到**的附加信息；在 `events` 里指定会被忽略。

关闭对某个 fd 的检查有两种写法：`events` 设为 0；或 `fd` 设为负值（`-fd`），此时 `events` 被忽略且 `revents` 总是 0。

实际编程最常用的就是 `POLLIN` 和 `POLLOUT`。

### 13.2.8 poll() 的返回值

与 `select()` 一致：

⚫ `-1`：出错并设置 `errno`；
⚫ `0`：超时，没有任何 fd 就绪；
⚫ `>0`：有 fd 就绪，值是 `fds` 数组中 `revents` **不为 0** 的元素个数。

### 13.2.9 使用示例

示例代码 13.2.3 就是把 13.2.1 的 `select` 换成 `poll`：

```c
struct pollfd fds[2];

fds[0].fd = 0;               /* 键盘 */
fds[0].events = POLLIN;      /* 只关心数据可读 */
fds[0].revents = 0;
fds[1].fd = fd;              /* 鼠标 */
fds[1].events = POLLIN;
fds[1].revents = 0;

while (loops--) {
    ret = poll(fds, 2, -1);
    if (0 > ret) { perror("poll error"); goto out; }
    else if (0 == ret) { fprintf(stderr, "poll timeout.\n"); continue; }

    if (fds[0].revents & POLLIN) {          /* 键盘就绪 */
        ret = read(0, buf, sizeof(buf));
        if (0 < ret) printf("键盘: 成功读取<%d>个字节数据\n", ret);
    }

    if (fds[1].revents & POLLIN) {          /* 鼠标就绪 */
        ret = read(fd, buf, sizeof(buf));
        if (0 < ret) printf("鼠标: 成功读取<%d>个字节数据\n", ret);
    }
}
```

判断方式是按位与：`revents & POLLIN`。

![图13.2.2 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-9%20%E5%9B%BE13.2.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

### 13.2.10 总结

⚠️ 用 `select()`/`poll()` 必须注意：**监测到某 fd 就绪后，一定要真正对它做一次 I/O，把这个就绪状态清掉**。否则状态一直挂着，下次调用会立即返回，程序变成死循环空转。

### select 与 poll 对比

| 维度 | select | poll |
|---|---|---|
| 传参形式 | 三个 `fd_set` | 一个 `struct pollfd` 数组 |
| fd 数量上限 | 受 `FD_SETSIZE`（1024）限制 | 无固定上限，由数组大小决定 |
| 超时单位 | `struct timeval`（秒 + 微秒） | 毫秒（`int`） |
| 每次调用前 | **必须重置并重新填充集合**（集合被改写） | 保持 `fds` 数组即可，`events` 不被改写 |
| 就绪判断 | `FD_ISSET()` | `revents & POLLIN` 等 |
| 额外信息 | 只有就绪与否 | `POLLERR`/`POLLHUP`/`POLLNVAL` 更明确 |
| 返回值 | 就绪 fd 个数（可能重复计数） | `revents` 非 0 的元素个数 |

---

## 13.3 异步 IO

I/O 多路复用是进程**主动查询**；异步 I/O 反过来——进程告诉内核"能 I/O 了给我发个信号"，然后去干别的事。所以异步 I/O 必须结合信号，也叫**信号驱动 I/O**。

使用步骤（5 步）：

⚫ 指定 `O_NONBLOCK` 使能非阻塞 I/O；
⚫ 指定 `O_ASYNC` 使能异步 I/O；
⚫ 设置异步 I/O 事件的接收进程（默认是调用进程）；
⚫ 为内核发送的通知信号注册信号处理函数——默认通知信号是 **`SIGIO`**；
⚫ 之后进程可执行其它任务，I/O 就绪时内核发 `SIGIO`，进程在处理函数里做 I/O。

### 13.3.1 O_ASYNC 标志

`O_ASYNC` 使能 fd 的异步 I/O 事件，fd 可 I/O 时内核向接收进程发 `SIGIO`。

⚠️ **`open()` 时指定 `O_ASYNC` 无法使能异步 I/O**，必须用 `fcntl()` 加：

```c
int flag;

flag = fcntl(fd, F_GETFL);
flag |= O_ASYNC;
fcntl(fd, F_SETFL, flag);
```

（原书示例第一行写成 `fcntl(0, F_GETFL)`，对鼠标 fd 应为 `fcntl(fd, F_GETFL)`。）

### 13.3.2 设置异步 I/O 事件的接收进程

用 `fcntl()` 的 `F_SETOWN` 命令，第三个参数传接收进程 PID，通常就是自己：

```c
fcntl(fd, F_SETOWN, getpid());
```

### 13.3.3 注册 SIGIO 信号的处理函数

用 `signal()` 或 `sigaction()` 为 `SIGIO` 注册处理函数，I/O 操作就放在里面做。

### 13.3.4 使用示例

示例代码 13.3.1：异步 I/O 读鼠标，`SIGIO` 到来时在 `sigio_handler()` 里 `read`。

```c
static int fd;

static void sigio_handler(int sig)
{
    static int loops = 5;
    char buf[100] = {0};
    int ret;

    if (SIGIO != sig) return;

    ret = read(fd, buf, sizeof(buf));
    if (0 < ret) printf("鼠标: 成功读取<%d>个字节数据\n", ret);

    loops--;
    if (0 >= loops) { close(fd); exit(0); }
}

int main(void)
{
    int flag;

    fd = open(MOUSE, O_RDONLY | O_NONBLOCK);    /* 非阻塞 */
    flag = fcntl(fd, F_GETFL);
    flag |= O_ASYNC;                            /* 使能异步 I/O */
    fcntl(fd, F_SETFL, flag);

    fcntl(fd, F_SETOWN, getpid());              /* 设置所有者 */
    signal(SIGIO, sigio_handler);               /* 注册 SIGIO */

    for ( ; ; ) sleep(1);
}
```

![图13.3.1 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-10%20%E5%9B%BE13.3.1%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

---

## 13.4 优化异步 I/O

需要同时检查**大量** fd（比如数千个）时，异步 I/O 相比 `select()`/`poll()` 有显著性能优势：内核能"记住"要检查的 fd，只在它们可 I/O 时发信号；而 `select()`/`poll()` 内部是轮询，fd 越多越费 CPU。fd 不多时，`select()`/`poll()` 反而是很好的方案。

> **Tips**：fd 很多时还可以用 epoll 解决 `select()`/`poll()` 性能低的问题，原书不展开；性能上 epoll 与异步 I/O 相似，但有些胜过异步 I/O 的优点。

13.3 的版本有两个缺陷：

⚫ **`SIGIO` 是非排队信号**。它是标准信号（非实时、不可靠），不支持排队：正在执行 `SIGIO` 处理函数时内核又发来多次 `SIGIO`，这些信号被阻塞，处理函数结束后**只能再传递一次**，其余全部丢失。
⚫ **无法得知 fd 发生了什么事件**。处理函数里直接 `read`，并不知道是可读、可写还是异常。

### 13.4.1 使用实时信号替换默认信号 SIGIO

同样用 `fcntl()`，命令换成 `F_SETSIG`，第三个参数给一个实时信号编号：

```c
fcntl(fd, F_SETSIG, SIGRTMIN);
```

之后 fd 可 I/O 时内核发 `SIGRTMIN` 而非 `SIGIO`。第三个参数传 **0** 表示恢复默认，即仍用 `SIGIO`。

### 13.4.2 使用 sigaction() 注册信号处理函数

为实时信号注册处理函数要用 `sigaction()`，并把 `sa_flags` 指定为 **`SA_SIGINFO`**——表示使用 `sa_sigaction` 指向的函数作为处理函数（而不是 `sa_handler`），它参数更多、能拿到更多信息。

处理函数的 `siginfo_t *` 由内核在触发信号时构建，异步 I/O 相关字段：

| 字段 | 含义 |
|---|---|
| `si_signo` | 引发处理函数被调用的信号，与第一个参数 `sig` 一致 |
| `si_fd` | 发生异步 I/O 事件的 fd |
| `si_code` | `si_fd` 上发生了什么事件（读就绪/写就绪/异常等） |
| `si_band` | 位掩码，值与 `poll()` 的 `revents` 相同，与 `si_code` 一一对应 |

表 13.4.1：

| `si_code` | `si_band` 掩码值 | 描述 |
|---|---|---|
| `POLL_IN` | `POLLIN \| POLLRDNORM` | 可读取数据 |
| `POLL_OUT` | `POLLOUT \| POLLWRNORM \| POLLWRBAND` | 可写入数据 |
| `POLL_MSG` | `POLLIN \| POLLRDNORM \| POLLMSG` | 不使用 |
| `POLL_ERR` | `POLLERR` | I/O 错误 |
| `POLL_PRI` | `POLLPRI \| POLLRDNORM` | 可读取高优先级数据 |
| `POLL_HUP` | `POLLHUP \| POLLERR` | 出现宕机 |

所以在处理函数里比对 `si_code` 就知道该做什么 I/O。

### 13.4.3 使用示例

示例代码 13.4.1 用"实时信号 + sigaction"同时解决两个缺陷。⚠️ 编译会报 `F_SETSIG` 未定义，需要在源文件**开头**定义 `_GNU_SOURCE`。

```c
#define _GNU_SOURCE            /* 必须在所有 include 之前 */

static int fd;

static void io_handler(int sig, siginfo_t *info, void *context)
{
    static int loops = 5;
    char buf[100] = {0};
    int ret;

    if (SIGRTMIN != sig) return;

    if (POLL_IN == info->si_code) {         /* 判断鼠标是否可读 */
        ret = read(fd, buf, sizeof(buf));
        if (0 < ret) printf("鼠标: 成功读取<%d>个字节数据\n", ret);

        loops--;
        if (0 >= loops) { close(fd); exit(0); }
    }
}

int main(void)
{
    struct sigaction act;
    int flag;

    fd = open(MOUSE, O_RDONLY | O_NONBLOCK);

    flag = fcntl(fd, F_GETFL);
    flag |= O_ASYNC;
    fcntl(fd, F_SETFL, flag);

    fcntl(fd, F_SETOWN, getpid());          /* 异步 I/O 所有者 */
    fcntl(fd, F_SETSIG, SIGRTMIN);          /* 换成实时信号 */

    act.sa_sigaction = io_handler;
    act.sa_flags = SA_SIGINFO;              /* 用 sa_sigaction 那个函数 */
    sigemptyset(&act.sa_mask);
    sigaction(SIGRTMIN, &act, NULL);

    for ( ; ; ) sleep(1);
}
```

![图13.4.1 编译报错](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-11%20%E5%9B%BE13.4.1%20%E7%BC%96%E8%AF%91%E6%8A%A5%E9%94%99%20%E6%8A%A5%E9%94%99%E6%8F%90%E7%A4%BA%E6%B2%A1%E6%9C%89%E5%AE%9A%E4%B9%89F_SETSIG%E7%A1%AE%E5%AE%9E%E5%A6%82.png)

![图13.4.2 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-12%20%E5%9B%BE13.4.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

---

## 13.5 存储映射 I/O

存储映射 I/O（memory-mapped I/O）把一个文件映射到进程地址空间的一块内存区域：**读这段内存就是读文件，写这段内存就是写文件**，完全不用 `read()`/`write()`。

![图13.5.1 存储映射 I/O 示意图](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-13%20%E5%9B%BE13.5.1%20%E5%AD%98%E5%82%A8%E6%98%A0%E5%B0%84%20IO%20%E7%A4%BA%E6%84%8F.png)

### 13.5.1 mmap() 和 munmap() 函数

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

| 参数 | 含义 |
|---|---|
| `addr` | 映射区起始地址。**通常传 `NULL`**，由系统选；非 `NULL` 只是自己建议一个地址 |
| `length` | 映射长度（字节），如 `1024 * 4` 表示映射 4K |
| `prot` | 映射区保护要求 |
| `flags` | 映射区属性，必须选 `MAP_SHARED` 或 `MAP_PRIVATE` 之一 |
| `fd` | 要映射的文件描述符 |
| `offset` | 文件内偏移，通常为 0（从文件头开始） |

`offset` + `length` 共同确定"文件的哪一段"被映射。

`prot` 取值（可用位或组合，也可为 `PROT_NONE`）：

⚫ `PROT_EXEC`：可执行；
⚫ `PROT_READ`：可读；
⚫ `PROT_WRITE`：可写；
⚫ `PROT_NONE`：不可访问。

⚠️ 保护要求**不能超过 `open()` 时的访问权限**：文件以只读打开，映射区就不能指定 `PROT_WRITE`。

`flags` 必须二选一：

⚫ **`MAP_SHARED`**：写入映射区的数据**会写回文件**，并允许其它进程共享；
⚫ **`MAP_PRIVATE`**：写时创建私人副本（copy-on-write），对映射区的操作**不更新文件**，只改副本。

可选标志（0 个或多个位或）：

| 标志 | 说明 |
|---|---|
| `MAP_FIXED` | 强制使用 `addr` 指定的地址，建不起来就放弃。不建议用，不利于移植 |
| `MAP_ANONYMOUS` | 匿名映射，忽略 `fd` 和 `offset`，不涉及文件，也无法与其它进程共享 |
| `MAP_ANON` | 与 `MAP_ANONYMOUS` 同义，不建议用 |
| `MAP_DENYWRITE` | 被忽略 |
| `MAP_EXECUTABLE` | 被忽略 |
| `MAP_FILE` | 兼容性标志，已忽略 |
| `MAP_LOCKED` | 对映射区域上锁 |

实际多数情况只指定 `MAP_SHARED`。

**返回值**：成功返回映射区起始地址；失败返回 `(void *)-1`，即 `MAP_FAILED`，并设置 `errno`。

**页对齐要求**：`addr` 和 `offset` 非 `NULL`/0 时，通常要求是**系统页大小的整数倍**。页大小用：

```c
sysconf(_SC_PAGE_SIZE)     /* 或 sysconf(_SC_PAGESIZE)，单位字节 */
```

`length` 没有这个限制，但实际情况是：**映射区大小总是页大小的整数倍**。文件 96 字节、`length` 传 96、页大小 4096 时，系统给 4096 字节映射区，后面 4000 字节补 0；改这 4000 字节**不会反映到文件**；访问 4000 字节之后的内存会异常，产生 `SIGBUS`。

⚠️ `length` **不能大于文件大小**，映射的部分不能超出文件。

### 13.5.2 与映射区相关的两个信号

⚫ **`SIGSEGV`**：映射区被指定为只读，进程却往里写，内核发此信号。默认动作是终止进程并生成 core dump。
⚫ **`SIGBUS`**：访问映射区时该部分已不存在——例如 `mmap` 时 `length` 取文件长度，访问前另一个进程把文件截断了（如 `ftruncate()`），再访问被截掉的部分就会收到 `SIGBUS`。默认动作同样是终止进程并生成 core dump。

### 13.5.3 munmap() 解除映射

`open()` 配 `close()`，`mmap()` 就配 `munmap()`：

```c
#include <sys/mman.h>

int munmap(void *addr, size_t length);
```

| 参数 | 含义 |
|---|---|
| `addr` | 待解除映射的起始地址，**必须是系统页大小的整数倍** |
| `length` | 待解除区域大小（字节），**同样按页大小对齐** |

两点注意：

⚫ **进程终止时会自动解除映射**（没显式 `munmap` 也没关系）；
⚫ **`close()` 关闭文件并不会解除映射**。

惯用法：`addr` 传 `mmap()` 的返回值、`length` 传 `mmap()` 的 `length`，即解除整个映射。

### 13.5.4 使用示例

示例代码 13.5.1：用存储映射 I/O 做文件复制（效果类似 `cp`）。

```c
struct stat sbuf;

srcfd = open(argv[1], O_RDONLY);
dstfd = open(argv[2], O_RDWR | O_CREAT | O_TRUNC, 0664);   /* 目标不存在则创建并清空 */

fstat(srcfd, &sbuf);                 /* 取源文件大小 */
ftruncate(dstfd, sbuf.st_size);      /* 目标文件撑到同样大小，这步不能省 */

srcaddr = mmap(NULL, sbuf.st_size, PROT_READ, MAP_SHARED, srcfd, 0);
if (MAP_FAILED == srcaddr) { perror("mmap error"); goto out2; }

dstaddr = mmap(NULL, sbuf.st_size, PROT_WRITE, MAP_SHARED, dstfd, 0);
if (MAP_FAILED == dstaddr) { perror("mmap error"); goto out3; }

memcpy(dstaddr, srcaddr, sbuf.st_size);        /* 一行完成复制 */

munmap(dstaddr, sbuf.st_size);
munmap(srcaddr, sbuf.st_size);
close(dstfd);
close(srcfd);
```

流程：源只读打开、目标读写 + 不存在则创建 + 截断为 0；`fstat()` 取源大小、`ftruncate()` 把目标撑到一样大（否则写映射区会 `SIGBUS`）；源映射 `PROT_READ`、目标映射 `PROT_WRITE`；`memcpy` 搬完；按后开先关的顺序回收。

![图13.5.2 源文件中的内容](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-14%20%E5%9B%BE13.5.2%20%E6%BA%90%E6%96%87%E4%BB%B6%E4%B8%AD%E7%9A%84%E5%86%85%E5%AE%B9%20%E7%9B%AE%E6%A0%87%E6%96%87%E4%BB%B6%20dstfile%20%E5%B9%B6%E4%B8%8D%E5%AD%98.png)

![图13.5.3 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-15%20%E5%9B%BE13.5.3%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E7%94%B1%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF.png)

### 13.5.5 mprotect() 函数

修改现有映射区的保护要求：

```c
#include <sys/mman.h>

int mprotect(void *addr, size_t len, int prot);
```

| 参数 | 含义 |
|---|---|
| `addr` | 地址范围起始，**必须是系统页大小的整数倍** |
| `len` | 地址范围大小 |
| `prot` | 取值同 `mmap()` 的 `prot` |

成功 0；失败 -1 并设置 `errno`。

### 13.5.6 msync() 函数

`write()` 写的数据不会立刻落盘，而是先进内核缓冲区，之后由内核在某个时刻刷盘——所以有"数据不同步"。映射区同理：写进映射区的数据也要等某个时刻才写回磁盘，映射区与磁盘文件会不一致。

`msync()` 用于把映射区数据**刷写/更新到磁盘**，类似 `fsync()`，只是作用于映射区。

```c
#include <sys/mman.h>

int msync(void *addr, size_t length, int flags);
```

| 参数 | 含义 |
|---|---|
| `addr` | 同步区域起始地址，**必须与系统页大小对齐** |
| `length` | 同步区域大小 |
| `flags` | `MS_ASYNC` 或 `MS_SYNC` 二选一，可选加 `MS_INVALIDATE` |

⚫ `MS_ASYNC`：异步同步，调用后不等数据写完磁盘就返回；
⚫ `MS_SYNC`：同步同步，等数据全部写入磁盘才返回；
⚫ `MS_INVALIDATE`：可选，请求使同一文件的其它映射失效（以便用刚写入的新值更新它们）。

成功 0；失败 -1 并设置 `errno`。

⚠️ **`munmap()` 不会把映射区内容写到磁盘**。`MAP_SHARED` 时文件的更新由内核按虚拟存储算法在某个时刻自动完成；`MAP_PRIVATE` 时解除映射后，进程对映射区的修改**直接丢弃**。

### 13.5.7 普通 I/O 与存储映射 I/O 比较

**普通 I/O 的缺点**：`read()`/`write()` 要经过层层函数调用才最终操作文件，数据在不同缓存间来回倒腾，效率偏低；标准 I/O（`fread()`/`fwrite()`）本质是对普通 I/O 的封装，同理。但数据量不大时影响很小，用起来反而方便。

**存储映射 I/O 的优点**：实质是**共享**，与 IPC 的共享内存很相似。以文件复制为例——

- 普通 I/O：先把源文件数据读进应用层缓冲区，再写进目标文件，**两次数据搬运**；
- 存储映射 I/O：两个文件都已在应用层内存里，`memcpy` 一次搞定，**少一次复制**，也少了大量函数调用。

![图13.5.4 普通 I/O 实现文件复制示例](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-16%20%E5%9B%BE13.5.4%20%E6%99%AE%E9%80%9A%20IO%20%E5%AE%9E%E7%8E%B0%E6%96%87%E4%BB%B6%E5%A4%8D%E5%88%B6%E7%A4%BA%E4%BE%8B.png)

![图13.5.5 存储映射 I/O 实现文件复制](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-17%20%E5%9B%BE13.5.5%20%E5%AD%98%E5%82%A8%E6%98%A0%E5%B0%84%20IO%20%E5%AE%9E%E7%8E%B0%E6%96%87%E4%BB%B6%E5%A4%8D%E5%88%B6%20%E9%A6%96%E5%85%88%E9%9D%9E%E5%B8%B8%E7%9B%B4%E8%A7%82%E7%9A%84%E4%B8%80%E7%82%B9.png)

"共享"怎么理解：应用层和内核层不能直接交互，必须走系统调用。而把文件直接映射到应用层地址空间后，磁盘文件与映射区直接关联，不用 `read`/`write` 就能互相反映——映射区可以看成**应用层与内核层之间的共享内存**。

**存储映射 I/O 的不足**：映射的文件只能是**固定大小**（`mmap` 时 `length` 就定死了）；映射区大小必须是页大小整数倍，96 字节的文件占 4096 字节、剩下 4000 字节补 0，能访问但改不动文件。所以它适合**大数据量**，少量数据用普通 I/O 更方便。

**应用场景**：视频图像处理这类大数据量场合，例如第二篇的 Framebuffer 编程（LCD 编程）就会用存储映射 I/O。

### 普通 I/O 与存储映射 I/O 对比

| 维度 | 普通 I/O（read/write） | 存储映射 I/O（mmap） |
|---|---|---|
| 数据路径 | 内核缓冲 → 用户缓冲 → 内核缓冲 → 文件 | 直接读写映射区，内核负责回写 |
| 数据复制/调用层次 | 复制多、函数调用层次深 | 少一次复制，效率高 |
| 适用数据量 | 小数据量更方便 | 大数据量优势明显 |
| 文件大小 | 可任意增长（`write` 追加） | 映射时固定，受 `length` 限制 |
| 对齐要求 | 无 | `addr`/`offset`，以及 `munmap`、`msync` 的 `addr` 需页对齐 |
| 同步 | `write` 后由内核刷盘，可用 `fsync` | 需 `msync` 主动刷，`munmap` 不刷盘 |
| 典型用途 | 一般文件读写 | LCD/Framebuffer、大文件处理 |

---

## 13.6 文件锁

两个进程同时编辑同一份文件会怎样？最终状态通常取决于**最后一个写它的进程**，中间内容很容易乱掉，因为并发 I/O 产生了竞争。

有些程序需要保证"这段时间只有我能操作这个文件"，Linux 提供文件锁。它与互斥锁/自旋锁/读写锁一样都是**内核提供的锁机制**，区别在场景：前几种用于**多线程**保护共享资源，文件锁用于**多进程**同时操作同一文件。

用法就是：操作文件前先加锁，读写完再解锁；没解锁期间别的进程动不了它。

### 13.6.1 文件锁的分类

⚫ **建议性锁**：本质是**协议**。大家约定访问前先加锁；但你要是硬不加锁直接读写，也照样能读写，锁拦不住。就像红灯不能通行，非要闯谁也拦不住——只有大家都遵守，锁才有意义。
⚫ **强制性锁**：强制生效。加了强制锁后，别的进程没拿到锁就**无法访问**文件。原因是内核会检查每一个 I/O 操作（`read()`/`write()`），验证调用进程是不是锁的拥有者；文件被加写锁时，内核阻止其它进程读写。代价是**每次读写都要查锁，性能影响很大**。

Linux 下可用 **`flock()`、`fcntl()`、`lockf()`** 三个函数加锁。

### 13.6.2 flock() 函数加锁

```c
#include <sys/file.h>

int flock(int fd, int operation);
```

| 参数 | 含义 |
|---|---|
| `fd` | 需要加锁的文件描述符 |
| `operation` | 操作方式，见下 |

`operation` 取值：

⚫ `LOCK_SH`：**共享锁**，多个进程可同时拥有同一文件的共享锁；
⚫ `LOCK_EX`：**排它锁**（互斥锁），同一时刻只能被一个进程拥有；
⚫ `LOCK_UN`：解锁、释放锁；
⚫ `LOCK_NB`：以**非阻塞**方式获取锁。默认拿不到锁会阻塞到别人释放；加了它拿不到就立即返回错误，`errno` 为 `EWOULDBLOCK`。通常与 `LOCK_SH`/`LOCK_EX` 位或组合。

**返回值**：成功 0；失败 -1 并设置 `errno`。

⚠️ 同一个文件**不会同时具有共享锁和互斥锁**。

### 13.6.3 使用示例

示例代码 13.6.1（进程 1）：非阻塞 + 排它锁加锁，然后注册 `SIGINT`，在信号处理函数里解锁退出。

```c
static int fd = -1;

static void sigint_handler(int sig)
{
    if (SIGINT != sig) return;

    flock(fd, LOCK_UN);                     /* 解锁 */
    close(fd);
    printf("进程 1: 文件已解锁!\n");
}

fd = open(argv[1], O_WRONLY);
if (-1 == fd) { perror("open error"); exit(-1); }

if (-1 == flock(fd, LOCK_EX | LOCK_NB)) {
    perror("进程 1: 文件加锁失败");
    exit(-1);
}

printf("进程 1: 文件加锁成功!\n");
signal(SIGINT, sigint_handler);

for ( ; ; ) sleep(1);                       /* 一直持锁 */
```

示例代码 13.6.2（进程 2）：**即使加锁失败也照样读写**，用来验证建议性锁的性质。

```c
fd = open(argv[1], O_RDWR);

if (-1 == flock(fd, LOCK_EX | LOCK_NB))
    perror("进程 2: 文件加锁失败");
else
    printf("进程 2: 文件加锁成功!\n");

write(fd, buf, len);                        /* 加锁失败也照写 */
lseek(fd, 0x0, SEEK_SET);
memset(buf, 0x0, sizeof(buf));
read(fd, buf, len);                         /* 照读 */

flock(fd, LOCK_UN);
close(fd);
```

两份代码分别编译成 `testApp1`、`testApp2`，`touch` 一个 `infile` 做测试文件。先把 `testApp1 infile` 放后台跑（`ps` 可见 pid），再跑 `testApp2 infile`：

![图13.6.1 两份可执行文件](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-18%20%E5%9B%BE13.6.1%20%E4%B8%A4%E4%BB%BD%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%20%E5%9C%A8%E8%BF%9B%E8%A1%8C%E6%B5%8B%E8%AF%95%E4%B9%8B%E5%89%8D%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E6%B5%8B%E8%AF%95%E7%94%A8%E7%9A%84%E6%96%87.png)

![图13.6.2 执行 testApp1](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-19%20%E5%9B%BE13.6.2%20%E6%89%A7%E8%A1%8C%20testApp1.png)

![图13.6.3 执行 testApp2](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-20%20%E5%9B%BE13.6.3%20%E6%89%A7%E8%A1%8C%20testApp2.png)

`testApp2` 加锁失败（锁被 `testApp1` 持有），但**读写照样成功**——这就是建议性锁。正确用法是：加锁失败后就不要再做 I/O，自觉遵守协议。

接着向 `testApp1` 发 `SIGINT`（`kill -2`）让它解锁退出，再跑一次 `testApp2`，这次加锁和读写都成功。

![图13.6.4 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-21%20%E5%9B%BE13.6.4%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

### 13.6.4 关于 flock() 的几条规则

⚫ **同一进程对文件多次加锁不会死锁**。再次 `flock()` 会**替换**旧锁：先加共享锁再加排它锁，最终是排它锁。
⚫ **文件关闭时自动解锁**。fd 关闭后锁自动释放；进程终止时它建立的所有锁全部释放。
⚫ **一个进程不能解另一个进程持有的锁**。
⚫ **`fork()` 出来的子进程不继承父进程的锁**。子进程虽然继承了 fd，但**不继承文件锁**，对父进程的锁而言它算另一个进程。这个约束合理：锁就是为了防止多进程同时写，能继承就失去意义了。
⚫ **fd 被复制时（`dup()`、`dup2()`、`fcntl` 的 `F_DUPFD`），复制出的 fd 与源 fd 引用同一个锁**，用任何一个解锁都行：

```c
flock(fd, LOCK_EX);          /* 加锁 */
new_fd = dup(fd);
flock(new_fd, LOCK_UN);      /* 用新 fd 也能解锁 */
```

但若不显式解锁，则要**所有 fd 都关闭**之后锁才释放。

### 13.6.5 fcntl() 函数加锁

`fcntl()` 是万能的文件描述符管理工具箱，靠不同 `cmd` 做不同事：

```c
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* struct flock *flockptr */ );
```

与锁相关的 `cmd`：**`F_SETLK`、`F_SETLKW`、`F_GETLK`**，第三个参数是 `struct flock *`。

与 `flock()` 的两大区别：

⚫ `flock()` 只能**对整个文件**加锁；`fcntl()` 可以**对文件的某个区域**加锁，精确到字节；
⚫ `flock()` 只能产生建议性锁；`fcntl()` **建议性锁和强制性锁都支持**。

### 13.6.6 struct flock 结构体

```c
struct flock {
    ...
    short l_type;      /* Type of lock: F_RDLCK, F_WRLCK, F_UNLCK */
    short l_whence;    /* How to interpret l_start: SEEK_SET, SEEK_CUR, SEEK_END */
    off_t l_start;     /* Starting offset for lock */
    off_t l_len;       /* Number of bytes to lock */
    pid_t l_pid;       /* PID of process blocking our lock (set by F_GETLK and F_OFD_GETLK) */
    ...
};
```

| 成员 | 含义 |
|---|---|
| `l_type` | 锁类型：`F_RDLCK` 共享读锁 / `F_WRLCK` 独占写锁 / `F_UNLCK` 解锁一个区域 |
| `l_whence`、`l_start` | 加锁/解锁区域的起始字节偏移，含义同 `lseek()` 的 `whence`/`offset` |
| `l_len` | 加锁/解锁区域的字节长度 |
| `l_pid` | 持有锁并能阻塞当前进程的那个进程的 pid，**仅 `cmd = F_GETLK` 时有效** |

区域规则：

⚫ 锁区域可以在**文件末尾处或越过末尾处**开始，但**不能在文件起始位置之前**开始；
⚫ `l_len` 为 **0** 表示"从起始位置一直到文件最大偏移量"，而且**是动态的**——之后往文件追加多少数据都落在锁区里；
⚫ 要锁**整个文件**：`l_whence = SEEK_SET`、`l_start = 0`、`l_len = 0`。

### 13.6.7 两种类型的锁：F_RDLCK 和 F_WRLCK

规则与 12.5 的线程读写锁很像：

⚫ 一个给定字节上可以有**多把共享读锁**；
⚫ 一个给定字节上只能有**一把独占写锁**；
⚫ 该字节上已有一把或多把读锁时，**不能加写锁**；
⚫ 该字节上已有写锁时，**不能再加任何锁**（读、写都不行）。

![图13.6.5 不同类型锁彼此之间的兼容性](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-22%20%E5%9B%BE13.6.5%20%E4%B8%8D%E5%90%8C%E7%B1%BB%E5%9E%8B%E9%94%81%E5%BD%BC%E6%AD%A4%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%BC%E5%AE%B9%E6%80%A7.png)

同一进程对已加锁区域再加锁，**新锁替换旧锁**：比如 100~200 字节已有写锁，又申请 100~200 的读锁，请求成功，写锁变成读锁。

权限要求：加**读锁**，进程对该文件必须有读权限（`O_RDONLY` 或 `O_RDWR`）；加**写锁**，必须有写权限（`O_WRONLY` 或 `O_RDWR`）。

### 13.6.8 F_SETLK、F_SETLKW 和 F_GETLK

| cmd | 作用 |
|---|---|
| `F_GETLK` | **测试**：`flockptr` 描述的锁能否加成功。若该区域已有别人持有的、与请求冲突的锁，则把那把现有锁的信息**写回** `flockptr`；若能加成功，则只把 `l_type` 改成 `F_UNLCK`，其它字段不变 |
| `F_SETLK` | **加锁**：按 `flockptr` 加读锁/写锁；失败立即返回错误，`errno` 为 `EACCES` 或 `EAGAIN`。`l_type = F_UNLCK` 时用于**解锁** |
| `F_SETLKW` | `F_SETLK` 的**阻塞版**（W = wait）：请求的锁被别人占着就阻塞，直到锁可用再唤醒 |

⚠️ `F_GETLK` 很少用。先 `F_GETLK` 测、再 `F_SETLK` 加，**这两步不是原子的**——测完到真正加锁之间，文件仍可能被别人锁住。

### 13.6.9 使用示例与测试

示例代码 13.6.4：`fcntl()` 给整个文件加写锁、写数据、再解锁。

```c
struct flock lock = {0};

fd = open(argv[1], O_WRONLY);

lock.l_type = F_WRLCK;        /* 独占性写锁 */
lock.l_whence = SEEK_SET;     /* 文件头部 */
lock.l_start = 0;
lock.l_len = 0;               /* 0 = 到文件末尾，即整个文件 */
if (-1 == fcntl(fd, F_SETLK, &lock)) { perror("加锁失败"); exit(-1); }

printf("对文件加锁成功!\n");
write(fd, buf, strlen(buf));

lock.l_type = F_UNLCK;        /* 解锁 */
fcntl(fd, F_SETLK, &lock);

close(fd);
```

示例代码 13.6.5：**同一进程对同一文件的不同区域加不同的锁**。先把文件截断到 1024 字节，100~200 加写锁，400~500 加读锁。

```c
struct flock wr_lock = {0};
struct flock rd_lock = {0};

fd = open(argv[1], O_RDWR);
ftruncate(fd, 1024);

wr_lock.l_type = F_WRLCK;
wr_lock.l_whence = SEEK_SET;
wr_lock.l_start = 100;
wr_lock.l_len = 100;
if (-1 == fcntl(fd, F_SETLK, &wr_lock)) { perror("加写锁失败"); exit(-1); }

rd_lock.l_type = F_RDLCK;
rd_lock.l_whence = SEEK_SET;
rd_lock.l_start = 400;
rd_lock.l_len = 100;
if (-1 == fcntl(fd, F_SETLK, &rd_lock)) { perror("加读锁失败"); exit(-1); }

/* 对文件进行 I/O 操作，完成后分别置 F_UNLCK 解锁 */
```

若两个区域重叠（如 100~200 和 150~250），重叠部分 150~200 会被**新锁覆盖**：先对 100~200 加写锁、再对 150~250 加读锁，则 150~200 最终由读锁控制。

**读锁共享性测试**（示例代码 13.6.6）：对 400~500 加读锁后死循环持锁，多跑几个实例——全部加读锁成功，证明**读锁共享**。

```c
lock.l_type = F_RDLCK;
lock.l_whence = SEEK_SET;
lock.l_start = 400;
lock.l_len = 100;
if (-1 == fcntl(fd, F_SETLK, &lock)) { perror("加读锁失败"); exit(-1); }

printf("加读锁成功!\n");
for ( ; ; ) sleep(1);
```

![图13.6.6 读锁共享性测试](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-23%20%E5%9B%BE13.6.6%20%E8%AF%BB%E9%94%81%E5%85%B1%E4%BA%AB%E6%80%A7%E6%B5%8B%E8%AF%95.png)

> 程序是后台跑的，测完用 `kill` 杀掉，或者关掉终端重开一个。

**写锁独占性测试**（示例代码 13.6.7）：同样结构，`l_type` 换成 `F_WRLCK`——只有第一个进程加锁成功，后面启动的都失败，证明**写锁独占**。

![图13.6.7 写锁的独占性测试](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-24%20%E5%9B%BE13.6.7%20%E5%86%99%E9%94%81%E7%9A%84%E7%8B%AC%E5%8D%A0%E6%80%A7%E6%B5%8B%E8%AF%95%20%E7%94%B1%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF.png)

### 13.6.10 几条规则

与 `flock()` 相似：

⚫ 文件关闭时自动解锁；
⚫ 一个进程不能解另一个进程持有的锁；
⚫ `fork()` 的子进程**不继承**父进程的锁；
⚫ fd 被复制（`dup`/`dup2`/`F_DUPFD`）时，复制出的 fd 与源 fd 引用**同一个锁**，用任一个解锁都行。

```c
lock.l_type = F_RDLCK;
fcntl(fd, F_SETLK, &lock);       /* 加锁 */

new_fd = dup(fd);

lock.l_type = F_UNLCK;
fcntl(new_fd, F_SETLK, &lock);   /* 用新 fd 解锁 */
```

⚠️ **与 `flock()` 的关键差异**：`fcntl()` 的锁在**任何一个** fd 被关闭后就自动释放（上例不显式解锁时，`fd` 或 `new_fd` 谁先关，锁就没了）；而 `flock()` 要**所有** fd 都关闭才释放。

### 13.6.11 建议性锁和强制性锁

`fcntl()` 两种都支持，但一般不建议用强制性锁，绝大多数场合用的是建议性锁。

开启强制锁机制靠**文件权限位**：把 **Set-Group-ID（`S_ISGID`）置 1**，同时**禁止组用户执行权限（`S_IXGRP` 置 0）**。

但有些 Linux/Unix 发行版根本不支持强制锁，可用代码测（示例代码 13.6.8）：先写一行数据、用 `fchmod` 开强制锁机制，`fork` 后父进程对整文件加写锁并 `wait`；子进程 `sleep(1)` 保证父先进、把文件设成非阻塞方式（**强制锁下读写会被阻塞，所以必须显式设非阻塞**）、尝试加读锁、再把偏移移到文件头 `read`。

```c
if (0 > fstat(fd, &sbuf)) { perror("fstat error"); exit(-1); }
if (0 > fchmod(fd, (sbuf.st_mode & ~S_IXGRP) | S_ISGID)) {
    perror("fchmod error"); exit(-1);
}
```

判断结论：

- 子进程加读锁失败（父进程持有写锁），这是必然的；
- 此时子进程 `read()` 若**失败**，说明系统支持强制性锁；若**成功**，说明系统不支持。

原书在 Ubuntu 上测的结果是：加读锁失败，但读文件成功——**Ubuntu 不支持强制性锁机制**。

![图13.6.8 Ubuntu 系统下测试结果](photo/%E7%AC%AC%E5%8D%81%E4%B8%89%E7%AB%A0%20%E9%AB%98%E7%BA%A7IO/13-25%20%E5%9B%BE13.6.8%20Ubuntu%20%E7%B3%BB%E7%BB%9F%E4%B8%8B%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

### 13.6.12 lockf() 函数加锁

`lockf()` 是一个**库函数**，内部基于 `fcntl()` 实现，是对 fcntl 锁的封装，用法原书不再展开。

---

## 13.7 小结

| 功能 | 关键接口 | 一句话 |
|---|---|---|
| 非阻塞 I/O | `open(..., O_NONBLOCK)`、`fcntl(F_SETFL)` | 进程发起 I/O 不被阻塞 |
| I/O 多路复用 | `select()`、`poll()` | 一个进程盯住多路 fd |
| 异步 I/O | `O_ASYNC` + `F_SETOWN` + `SIGIO`/`F_SETSIG` | fd 可 I/O 时内核发信号通知 |
| 存储映射 I/O | `mmap()`/`munmap()`/`mprotect()`/`msync()` | 文件当内存用 |
| 文件锁 | `flock()`、`fcntl()`、`lockf()` | 多进程互斥访问文件 |

## 本章速记

| 要点 | 结论 |
|---|---|
| 谁能阻塞 | 管道、网络设备、字符设备可阻塞；**普通文件永远不阻塞** |
| 切非阻塞 | `open` 加 `O_NONBLOCK`；已打开的 fd 用 `fcntl(F_GETFL/F_SETFL)` 改 |
| 非阻塞读失败 | `errno` 为 `EAGAIN`，`perror` 打印 `Resource temporarily unavailable` |
| 阻塞 vs 非阻塞 | 阻塞省 CPU 但卡死一路；非阻塞能并发但轮询吃满 CPU |
| select 的 nfds | 三个集合中**最大 fd + 1** |
| select 超时 | `NULL` 永久阻塞；`timeval` 全 0 立即返回；否则为上限 |
| **select 的坑** | 返回时集合被改写，**每轮都要 `FD_ZERO` + 重新 `FD_SET`** |
| select 返回 0 | 超时，三个集合被清空（不是出错） |
| FD_SETSIZE | 1024，fd 集合容量上限 |
| 就绪必须处理 | 就绪后不做 I/O 清除状态，下次调用立即返回，变成死循环 |
| poll 超时 | `-1` 永久、`0` 立即、`>0` 毫秒 |
| poll 判断 | `revents & POLLIN`；`events` 不会被 poll 改写 |
| 关闭某个 fd 的检查 | `events = 0` 或 `fd = -fd` |
| 异步 I/O 五步 | `O_NONBLOCK` → `O_ASYNC` → `F_SETOWN` → 注册信号 → 等信号 |
| O_ASYNC 的坑 | **不能靠 `open` 指定**，只能用 `fcntl(F_SETFL)` 加 |
| SIGIO 两个缺陷 | 非实时信号不排队会丢；无法得知发生了什么事件 |
| 优化手段 | `fcntl(fd, F_SETSIG, SIGRTMIN)`；`sigaction` + `SA_SIGINFO` |
| si_code | `POLL_IN`/`POLL_OUT`/`POLL_ERR`/`POLL_HUP`/`POLL_PRI`，与 `si_band` 对应 |
| F_SETSIG 编译报错 | 源文件开头 `#define _GNU_SOURCE` |
| mmap 返回 | 成功为映射区首地址；失败为 `MAP_FAILED`（`(void *)-1`） |
| mmap 页对齐 | `addr`/`offset` 需页对齐；`length` 不要求，但实际按页分配 |
| 页大小 | `sysconf(_SC_PAGE_SIZE)` |
| length 限制 | **不能大于文件大小**，越界访问收 `SIGBUS` |
| MAP_SHARED vs PRIVATE | 写回文件 / 写时复制，只改副本 |
| prot 限制 | 不能超过 `open` 时的访问权限（只读打开不能 `PROT_WRITE`） |
| 两个信号 | 写只读映射区 → `SIGSEGV`；访问已被截断部分 → `SIGBUS` |
| munmap | `addr` 必须页对齐；`close` **不解除**映射；进程退出自动解除 |
| msync | `MS_ASYNC`/`MS_SYNC`，可选 `MS_INVALIDATE`；`munmap` 不刷盘 |
| mmap 复制前提 | 目标文件先 `ftruncate` 到源文件大小，否则写映射区 `SIGBUS` |
| flock 锁类型 | `LOCK_SH`/`LOCK_EX`/`LOCK_UN`/`LOCK_NB`（非阻塞，`EWOULDBLOCK`） |
| flock 规则 | 同进程重复加锁是替换不死锁；关闭 fd 自动解锁；不能解别人的锁；fork 不继承 |
| fcntl 三命令 | `F_GETLK` 测试、`F_SETLK` 非阻塞加/解、`F_SETLKW` 阻塞加 |
| struct flock | `l_type`、`l_whence`、`l_start`、`l_len`、`l_pid`（仅 GETLK 有效） |
| 锁整个文件 | `l_whence = SEEK_SET`、`l_start = 0`、`l_len = 0` |
| l_len = 0 | 从起始位置到文件末尾，且**动态**跟随追加 |
| fcntl 锁继承 | fork 不继承；dup 共享同一把锁，**任一 fd 关闭即释放**（与 flock 不同） |
| 强制锁开关 | `S_ISGID = 1` 且 `S_IXGRP = 0`；Ubuntu 不支持 |
| lockf | 库函数，基于 `fcntl` 封装 |
