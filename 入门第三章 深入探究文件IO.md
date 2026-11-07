# 第三章 深入探究文件 I/O

> 上一章只教会了“怎么用”这四个系统调用；本章回答“背后发生了什么”。会 `open`/`read`/`write` 只能写简单的读写程序，一旦遇到**追加写错位、数据被覆盖、文件被创建两次、程序退出丢数据**这类问题，就必须知道 inode、文件表、动态文件、原子操作这些底层机制才能定位。
>
> 本章主题：Linux 如何管理文件（inode + 文件表 + 动态文件）、错误处理 `errno`/`strerror`/`perror`、三种退出方式、空洞文件、`O_TRUNC`/`O_APPEND`、多次打开同一文件、`dup`/`dup2` 复制 fd、文件共享、原子操作与竞争冒险、`fcntl`/`ioctl`、`truncate`/`ftruncate`。

一句话主线：**fd 是进程内的索引 → 文件表记录“怎么打开、读到哪” → inode 记录“文件本身长什么样、数据在哪”**。所有“诡异现象”都是这条链上某一层没搞清。

---

## 3.1 Linux 系统如何管理文件

### 3.1.1 静态文件与 inode

文件没被打开时躺在磁盘里，这种形态叫**静态文件**。

磁盘最小存储单位是**扇区（Sector）**，每扇区 512 字节（0.5KB）。操作系统不会一个扇区一个扇区地读（太慢），而是一次连续读多个扇区，即一个**块（block）**；块是文件存取的最小单位，最常见 **4KB**（连续 8 个 sector）。

磁盘分区、格式化时分成两个区域：

| 区域 | 作用 |
|---|---|
| 数据区 | 存文件真正的内容数据 |
| inode 区 | 存 inode table（inode 表），表里是一个个 inode 节点 |

**每个文件必须对应唯一一个 inode**，不同 inode 代表不同文件。inode 实质是一个结构体，记录：文件字节大小、文件所有者、读/写/执行权限、时间戳（创建/修改）、文件类型、**文件数据所在的 block 位置**等。

⚫ **文件名并不记录在 inode 中**（文件名存在目录项里，见第五章）。
⚫ inode table 本身也占磁盘空间，所以格式化会消耗容量。

![图3.1.1 inode table 与 inode](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-1%20%E5%9B%BE3.1.1%20inode%20table%20%E4%B8%8E%20inode.png)

每个 inode 有唯一数字编号，通过编号可在 inode table 中定位。查看方式：

```bash
ls -i test_file        # 第一列就是 inode 编号
stat test_file         # 输出中 Inode: 一行
```

![图3.1.2 ls 查看文件的 inode 编号](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-2%20%E5%9B%BE3.1.2%20ls%20%E6%9F%A5%E7%9C%8B%E6%96%87%E4%BB%B6%E7%9A%84%20inode%20%E7%BC%96%E5%8F%B7.png)

![图3.1.3 stat 查看 inode 编号](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-3%20%E5%9B%BE3.1.3%20stat%20%E6%9F%A5%E7%9C%8B%20inode%20%E7%BC%96%E5%8F%B7.png)

Windows 的“快速格式化”只清 inode table，真正的数据区没动，所以格式化后的数据仍可能被找回；普通格式化才会真正慢。

![图3.1.4 Windows 下格式化磁盘](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-4%20%E5%9B%BE3.1.4%20Windows%20%E4%B8%8B%E6%A0%BC%E5%BC%8F%E5%8C%96%E7%A3%81%E7%9B%98.png)

打开一个文件，系统内部三步走：

1. 由文件名找到对应的 inode 编号；
2. 用编号从 inode table 中找到 inode 结构体；
3. 按 inode 里记录的 block 位置把数据读出来。

### 3.1.2 文件打开时的状态（动态文件、文件表、fd 表）

调用 `open` 时，内核申请一段内存缓冲区，把静态文件的数据从磁盘读进内存管理、缓存，这份内存中的文件数据叫**动态文件**（也就是**内核缓冲区**）。之后的读写全部针对动态文件，不直接针对磁盘上的静态文件。

由此产生两个日常现象，也就都解释得通了：打开大文件慢（要把数据搬进内存）；文档写一半掉电，内容丢失（动态文件还没同步回磁盘）。同步由内核在之后完成。

为什么这么设计：磁盘/硬盘/U盘都是 **Flash 块设备**，只能以块为单位读写，改一个字节也要把整块读出、改、再写回，极不灵活；内存可按字节随机访问，且速率远高于磁盘。所以先缓存到内存再操作。

Linux 内核为每个进程设一个数据结构管理其状态，叫**进程控制块 PCB（Process Control Block）**。PCB 里有一个指针指向**文件描述符表（File descriptors）**；表中每个元素索引到一个**文件表（File table）**；文件表记录：**文件状态标志、引用计数、当前文件读写偏移量、i-node 指针**。

![图3.1.5 文件描述符表、文件表以及 inode 之间的关系](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-5%20%E5%9B%BE3.1.5%20%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E8%A1%A8%E6%96%87%E4%BB%B6%E8%A1%A8%E4%BB%A5%E5%8F%8A%20inode%20%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png)

记住这张图的层级：

```
PCB → 文件描述符表[fd] → 文件表（状态标志/引用计数/偏移量/i-node指针） → inode → 磁盘 block
```

**偏移量存在“文件表”里，不在 fd 里，也不在 inode 里** —— 这句话是理解 3.6、3.7 全部现象的钥匙。

---

## 3.2 返回错误处理与 errno

### 3.2.1 errno 变量

函数失败只返回 -1，我们并不知道原因。Linux 给常见错误编了号，出错时操作系统把编号写入 **`errno`** 变量。每个进程维护自己的 `errno`，它是程序中的全局变量，存储**最近一次**函数执行错误编号——下一次错误会覆盖上一次。

⚫ 本质是 `int` 类型变量。
⚫ **并非所有系统调用/C 库函数出错都会设置 errno**。判断方法：`man 2 open`，看 RETURN VALUE 段是否写了 “errno is set”。

![图3.2.1 查看返回值描述信息](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-6%20%E5%9B%BE3.2.1%20%E6%9F%A5%E7%9C%8B%E8%BF%94%E5%9B%9E%E5%80%BC%E6%8F%8F%E8%BF%B0%E4%BF%A1%E6%81%AF.png)

使用只需包含 `<errno.h>`：

```c
#include <stdio.h>
#include <errno.h>

int main(void)
{
    printf("%d\n", errno);
    return 0;
}
```

⚫ 读取 `errno` 要**紧跟**失败的那次调用，中间任何一次库函数调用都可能把它改掉。

### 3.2.2 strerror 函数

`errno` 只是个数字，`strerror()` 把它翻译成可读字符串。C 库函数（不是系统调用），`man 3 strerror`。

```c
#include <string.h>

char *strerror(int errnum);
```

| 参数 | 含义 |
|---|---|
| `errnum` | 错误编号，通常传 `errno` |

**返回值**：指向该错误编号对应描述字符串的指针。

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main(void)
{
    int fd;

    fd = open("./test_file", O_RDONLY);
    if (-1 == fd) {
        printf("Error: %s\n", strerror(errno));
        return -1;
    }

    close(fd);
    return 0;
}
```

文件不存在时打印 `Error: No such file or directory`。

![图3.2.2 strerror 测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-7%20%E5%9B%BE3.2.2%20strerror%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

### 3.2.3 perror 函数

`perror()` 直接把错误信息打印到标准错误，不用自己取 `errno`，也不用 `printf`。`man 3 perror`。

```c
#include <stdio.h>

void perror(const char *s);
```

| 参数 | 含义 |
|---|---|
| `s` | 自定义前缀信息；不需要就传空字符串 `""` |

**返回值**：无（`void`）。

```c
fd = open("./test_file", O_RDONLY);
if (-1 == fd) {
    perror("open error");
    return -1;
}
```

输出：`open error: No such file or directory` —— perror 会自动在前缀后补上**冒号 + 空格**。

![图3.2.3 perror 测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-8%20%E5%9B%BE3.2.3%20perror%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

> 两个都能用，实际工程里 `perror` 用得更多，推荐它。

---

## 3.3 exit、_exit、_Exit

函数出错且后续步骤无法继续时，应终止程序而不是硬着头皮往下跑。Linux 下进程退出分**正常退出**和**异常退出**（异常多指不可预料的系统异常、收到信号等），这里只讲正常退出。

正常退出有三种方式：

| 方式 | 类型 | 头文件 | 原型 |
|---|---|---|---|
| `return` | 语言关键字 | — | 从 main 返回，把控制权交回调用者，结束进程 |
| `_exit()` | **系统调用** | `<unistd.h>` | `void _exit(int status);` |
| `_Exit()` | **系统调用** | `<stdlib.h>` | `void _Exit(int status);` |
| `exit()` | **标准 C 库函数** | `<stdlib.h>` | `void exit(int status);` |

`_exit()` 会清除进程使用的内存空间、销毁内核中的各种数据结构、关闭所有文件描述符，结束进程并把控制权交给操作系统。`status` 为 0 表示正常结束，其它值表示执行中检测到错误。

```c
#include <unistd.h>
_exit(-1);       /* 出错退出 */
_exit(0);        /* 正常退出 */
```

`_exit()` 与 `_Exit()` **等价**，用法作用完全一样，都是系统调用。

`exit()` 是标准 C 库函数，它会**先做清理工作**（最重要的是**刷新 stdio 缓冲区**、调用 `atexit` 注册的函数），最后才调用 `_exit()`。

⚫ 这是本章第一个“后面会反复用到”的坑：**`_exit()` 不刷新 stdio 缓冲区**，`printf` 的内容可能丢失；`exit()` 会刷新。第四章讲缓冲区时会再撞上它。
⚫ 三者都能结束进程，日常推荐 `exit()`。

---

## 3.4 空洞文件

### 3.4.1 概念

`lseek()` 不仅能改偏移量，还**允许偏移量超出文件当前长度**。例如 `test_file` 大小 4KB（4096 字节），`lseek` 把偏移移到 6000 字节处是合法的；此时 `write()` 就从 6000 字节处开始写，于是 **4096~6000 字节之间没有任何数据，形成一个“空洞”**，这个文件就叫**空洞文件（hole file）**。

⚫ 空洞部分**不占任何物理空间**，直到某时刻对该区域写入数据才分配。
⚫ 但**逻辑上文件大小包含空洞**：`ls -l` 看到的是逻辑大小，`du` 看到的是实际占用块大小。

用途：多线程分段构建大文件（类似多个施工队分段修同一条高速）。现实场景：

- 迅雷多线程下载，未下完文件已占满全部大小的空间——就是空洞文件，各线程从不同偏移同时写；
- 创建虚拟机分配 100G 磁盘，实际只用 3~4G，先占位后按需分配。

### 3.4.2 实验

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(void)
{
    int fd;
    int ret;
    char buffer[1024];
    int i;

    /* 新建文件，新建文件大小为 0 */
    fd = open("./hole_file", O_WRONLY | O_CREAT | O_EXCL,
              S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
    if (-1 == fd) {
        perror("open error");
        exit(-1);
    }

    /* 偏移量直接跳到 4K 处，制造 0~4096 的空洞 */
    ret = lseek(fd, 4096, SEEK_SET);
    if (-1 == ret) {
        perror("lseek error");
        goto err;
    }

    memset(buffer, 0xFF, sizeof(buffer));

    /* 循环写 4 次，每次 1K，共写入 4K 真实数据 */
    for (i = 0; i < 4; i++) {
        ret = write(fd, buffer, sizeof(buffer));
        if (-1 == ret) {
            perror("write error");
            goto err;
        }
    }

    ret = 0;
err:
    close(fd);
    exit(ret);
}
```

![例3.4.2 空洞文件测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-9%20%E4%BE%8B3.4.2%20%E7%A9%BA%E6%B4%9E%E6%96%87%E4%BB%B6%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

结果：`ls -l` 显示 8K（逻辑大小 = 4K 空洞 + 4K 数据），`du` 显示 4K（实际占用）。

> 留个问题：用 `read` 读空洞部分会读到什么？（答案与 `ftruncate` 扩展部分一致：空字节 `\0`。）

---

## 3.5 O_APPEND 和 O_TRUNC 标志

### 3.5.1 O_TRUNC

作用极简：`open` 时带上它，**文件原有内容全部丢弃，大小变为 0**。

```c
fd = open("./test_file", O_WRONLY | O_TRUNC);
if (-1 == fd) {
    perror("open error");
    exit(-1);
}
close(fd);
```

只打开再关闭，不写任何数据。测试前 `test_file` 有 8760 字节，运行后变成 0 字节。

![图3.5.1 O_TRUNC 测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-10%20%E5%9B%BE3.5.1%20O_TRUNC%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

⚫ `O_TRUNC` 必须配合可写打开（`O_WRONLY`/`O_RDWR`），对只读打开无意义/会失败。

### 3.5.2 O_APPEND

带 `O_APPEND` 时，**每次 `write` 都会自动把写位置偏移量移到文件末尾**，从末尾写。

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void)
{
    char buffer[16];
    int fd;
    int ret;

    fd = open("./test_file", O_RDWR | O_APPEND);
    if (-1 == fd) {
        perror("open error");
        exit(-1);
    }

    memset(buffer, 0x55, sizeof(buffer));

    /* 写入 4 字节 0x55 */
    ret = write(fd, buffer, 4);
    if (-1 == ret) {
        perror("write error");
        goto err;
    }

    memset(buffer, 0x00, sizeof(buffer));

    /* 移到距文件末尾 4 字节处 */
    ret = lseek(fd, -4, SEEK_END);
    if (-1 == ret) {
        perror("lseek error");
        goto err;
    }

    ret = read(fd, buffer, 4);
    if (-1 == ret) {
        perror("read error");
        goto err;
    }

    printf("0x%x 0x%x 0x%x 0x%x\n", buffer[0], buffer[1], buffer[2], buffer[3]);
    ret = 0;
err:
    close(fd);
    exit(ret);
}
```

读回的 4 字节全是 `0x55`，证明写入确实落在了文件末尾。

![图3.5.2 O_APPEND 标志测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-11%20%E5%9B%BE3.5.2%20O_APPEND%20%E6%A0%87%E5%BF%97%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

两个必须记住的细节：

⚫ **`O_APPEND` 不影响读**。即使带了它，读位置偏移量默认仍在文件头。
⚫ **`O_APPEND` 下 `lseek` 改不了写位置**。即使用 `lseek` 把偏移移到别处，`write` 依然写末尾——因为 `write` 内部检测到 `O_APPEND` 就自动把写偏移移到末尾。

> 自测：`O_APPEND | O_TRUNC` 同时用会怎样？（先清空成 0 字节，然后每次写都追加到新末尾。）

---

## 3.6 多次打开同一个文件

同一个文件可以被多次打开——同一进程内多次打开、多个进程分别打开，都允许。三条结论：

### 3.6.1 一个进程内多次 open，得到多个不同 fd

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    int fd1, fd2, fd3;
    int ret;

    fd1 = open("./test_file", O_RDWR);
    if (-1 == fd1) { perror("open error"); exit(-1); }

    fd2 = open("./test_file", O_RDWR);
    if (-1 == fd2) { perror("open error"); ret = -1; goto err1; }

    fd3 = open("./test_file", O_RDWR);
    if (-1 == fd3) { perror("open error"); ret = -1; goto err2; }

    printf("%d %d %d\n", fd1, fd2, fd3);

    close(fd3);
    ret = 0;
err2:
    close(fd2);
err1:
    close(fd1);
    exit(ret);
}
```

结果：`6 7 8`。任何一个 fd 都能操作该文件，但**打开时用什么权限，拿到的 fd 就只有什么权限**；用完必须逐个 `close`。

![图3.6.1 打印文件描述符](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-12%20%E5%9B%BE3.6.1%20%E6%89%93%E5%8D%B0%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6.png)

⚫ 最小 fd 是 6 而不是 3，是因为程序在 vscode 的终端里跑，3/4/5 已被 vscode 进程占用；**子进程继承父进程的 fd**。直接在 Ubuntu 自带 Terminal 里跑就是 3、4、5。

### 3.6.2 内存中只有一份动态文件

```c
int main(void)
{
    char buffer[4];
    int fd1, fd2;
    int ret;

    fd1 = open("./test_file", O_RDWR | O_CREAT | O_EXCL,
               S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
    if (-1 == fd1) { perror("open error"); exit(-1); }

    fd2 = open("./test_file", O_RDWR);
    if (-1 == fd2) { perror("open error"); ret = -1; goto err1; }

    buffer[0] = 0x11; buffer[1] = 0x22; buffer[2] = 0x33; buffer[3] = 0x44;

    ret = write(fd1, buffer, 4);          /* 通过 fd1 写入 */
    if (-1 == ret) { perror("write error"); goto err2; }

    ret = lseek(fd2, 0, SEEK_SET);        /* fd2 回到文件头 */
    if (-1 == ret) { perror("lseek error"); goto err2; }

    memset(buffer, 0x00, sizeof(buffer));
    ret = read(fd2, buffer, 4);           /* 通过 fd2 读 */
    if (-1 == ret) { perror("read error"); goto err2; }

    printf("0x%x 0x%x 0x%x 0x%x\n", buffer[0], buffer[1], buffer[2], buffer[3]);

    ret = 0;
err2:
    close(fd2);
err1:
    close(fd1);
    exit(ret);
}
```

读回 `0x11 0x22 0x33 0x44` → 内存里只有**一份**动态文件，多次 open 不会拷多份。

![图3.6.2 测试结果 2](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-13%20%E5%9B%BE3.6.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%202.png)

### 3.6.3 不同 fd 的读写偏移量相互独立

偏移量记录在**文件表**中，多次 open 得到多个文件表，所以各自偏移独立；但所有文件表的 i-node 指针都指向**同一个 inode**。

![图3.6.3 多次打开同一个文件--文件描述符表文件表以及 inode 之间的关系](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-14%20%E5%9B%BE3.6.3%20%E5%A4%9A%E6%AC%A1%E6%89%93%E5%BC%80%E5%90%8C%E4%B8%80%E4%B8%AA%E6%96%87%E4%BB%B6--%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E8%A1%A8%E6%96%87%E4%BB%B6%E8%A1%A8%E4%BB%A5%E5%8F%8A%20i.png)

验证方法：把 3.6.2 代码里的 `lseek(fd2, 0, SEEK_SET)` 去掉，仍能读到 `0x11 0x22 0x33 0x44`，说明 fd2 的偏移仍为 0，与 fd1 无关。

> 多个**进程**打开磁盘上同一文件，内存中也只维护一份动态文件被多进程共享，各进程仍有独立的读写偏移量。
>
> 动态文件何时关闭？**引用计数为 0 时**系统自动关闭。文件表里的引用计数记录当前文件被多少个 fd 关联。

### 3.6.4 多次打开同一文件写操作：分别写

```c
int main(void)
{
    unsigned char buffer1[4], buffer2[4];
    int fd1, fd2;
    int ret, i;

    fd1 = open("./test_file", O_RDWR | O_CREAT | O_EXCL,
               S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
    if (-1 == fd1) { perror("open error"); exit(-1); }

    fd2 = open("./test_file", O_RDWR);
    if (-1 == fd2) { perror("open error"); ret = -1; goto err1; }

    buffer1[0] = 0x11; buffer1[1] = 0x22; buffer1[2] = 0x33; buffer1[3] = 0x44;
    buffer2[0] = 0xAA; buffer2[1] = 0xBB; buffer2[2] = 0xCC; buffer2[3] = 0xDD;

    for (i = 0; i < 4; i++) {
        write(fd1, buffer1, sizeof(buffer1));
        write(fd2, buffer2, sizeof(buffer2));
    }

    lseek(fd1, 0, SEEK_SET);

    for (i = 0; i < 8; i++) {
        ret = read(fd1, buffer1, sizeof(buffer1));
        if (-1 == ret) { perror("read error"); goto err2; }
        printf("%x%x%x%x", buffer1[0], buffer1[1], buffer1[2], buffer1[3]);
    }
    printf("\n");

    ret = 0;
err2:
    close(fd2);
err1:
    close(fd1);
    exit(ret);
}
```

结果打印 `aabbccddaabbccdd...`：两个 fd 从各自偏移写，fd2 的数据**覆盖**了 fd1 写的（因为它们都从 0 开始，偏移各自推进）。

![图3.6.4 测试结果 3](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-15%20%E5%9B%BE3.6.4%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%203.png)

### 3.6.5 加 O_APPEND 变成接续写

只把两个 `open` 都加上 `O_APPEND`，其余不变，输出变为 `11223344aabbccdd...`，即分别写 → **接续写**。

![图3.6.5 测试结果 4](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-16%20%E5%9B%BE3.6.5%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%204.png)

⚫ 用 `lseek` 在每次写之前移到末尾也能实现接续写，但**存在竞态**（见 3.9），多进程下不可靠；`O_APPEND` 才是原子方案。

---

## 3.7 复制文件描述符

`open` 返回的 fd 可以被复制，得到的新 fd 与旧 fd 都能操作同一文件，且**权限完全相同**（旧 fd 有读写权限，新 fd 也有）。两个系统调用：`dup`、`dup2`。

复制得到的 fd 与旧 fd **指向同一个文件表**：

![图3.7.1 指向同一个文件表 因为复制得到的文件描述符与旧的](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-17%20%E5%9B%BE3.7.1%20%E6%8C%87%E5%90%91%E5%90%8C%E4%B8%80%E4%B8%AA%E6%96%87%E4%BB%B6%E8%A1%A8%20%E5%9B%A0%E4%B8%BA%E5%A4%8D%E5%88%B6%E5%BE%97%E5%88%B0%E7%9A%84%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E4%B8%8E%E6%97%A7%E7%9A%84.png)

所以两者共享：文件读写权限、文件状态标志、**文件偏移量**。所谓“复制 fd”，实质是**复制文件表引用**——不是新建文件表。用完同样要 `close`。

### 3.7.1 dup

```c
#include <unistd.h>

int dup(int oldfd);
```

| 参数 | 含义 |
|---|---|
| `oldfd` | 需要被复制的文件描述符 |

**返回值**：成功返回**新的 fd**，由系统分配，遵循“最小可用 fd”原则；失败返回 -1 并设置 `errno`。

因为共享同一文件表（同一偏移量），不做 `O_APPEND` 也能实现接续写：

```c
int main(void)
{
    unsigned char buffer1[4], buffer2[4];
    int fd1, fd2;
    int ret, i;

    fd1 = open("./test_file", O_RDWR | O_CREAT | O_EXCL,
               S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
    if (-1 == fd1) { perror("open error"); exit(-1); }

    fd2 = dup(fd1);                 /* 复制 fd */
    if (-1 == fd2) { perror("dup error"); ret = -1; goto err1; }

    printf("fd1: %d\nfd2: %d\n", fd1, fd2);

    buffer1[0] = 0x11; buffer1[1] = 0x22; buffer1[2] = 0x33; buffer1[3] = 0x44;
    buffer2[0] = 0xAA; buffer2[1] = 0xBB; buffer2[2] = 0xCC; buffer2[3] = 0xDD;

    for (i = 0; i < 4; i++) {
        write(fd1, buffer1, sizeof(buffer1));
        write(fd2, buffer2, sizeof(buffer2));
    }

    lseek(fd1, 0, SEEK_SET);

    for (i = 0; i < 8; i++) {
        ret = read(fd1, buffer1, sizeof(buffer1));
        if (-1 == ret) { perror("read error"); goto err2; }
        printf("%x%x%x%x", buffer1[0], buffer1[1], buffer1[2], buffer1[3]);
    }
    printf("\n");

    ret = 0;
err2:
    close(fd2);
err1:
    close(fd1);
    exit(ret);
}
```

结果为接续写 `11223344aabbccdd...`，fd1 = 6、fd2 = 7。

![图3.7.2 dup 测试结果 由打印信息](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-18%20%E5%9B%BE3.7.2%20dup%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E7%94%B1%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF.png)

⚫ `dup` 的缺陷：新 fd 由系统分配，**不能自己指定**。

### 3.7.2 dup2

```c
#include <unistd.h>

int dup2(int oldfd, int newfd);
```

| 参数 | 含义 |
|---|---|
| `oldfd` | 需要被复制的文件描述符 |
| `newfd` | 手动指定的新 fd（应指定一个当前进程**未使用**的 fd） |

**返回值**：成功返回 `newfd` 本身；失败返回 -1 并设置 `errno`。

```c
fd1 = open("./test_file", O_RDWR | O_CREAT | O_EXCL,
           S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
if (-1 == fd1) { perror("open error"); exit(-1); }

fd2 = dup2(fd1, 100);       /* 指定新 fd 为 100 */
if (-1 == fd2) { perror("dup error"); ret = -1; goto err1; }

printf("fd1: %d\nfd2: %d\n", fd1, fd2);   /* fd2 == 100 */
```

![图3.7.3 dup2 函数测试结果 由打印信息](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-19%20%E5%9B%BE3.7.3%20dup2%20%E5%87%BD%E6%95%B0%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E7%94%B1%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF.png)

⚫ 同一个 fd 可以被 `dup`/`dup2` **复制多次**（如标准输出的重定向就是靠 `dup2`）。
⚫ `dup2` 若 `newfd` 已被占用，会先关闭它再复用——这既是能力也是坑。

---

## 3.8 文件共享

**文件共享**：同一个文件（磁盘上同一 inode）被多个**独立读写体**同时进行 IO 操作。“独立读写体”可理解为对应同一文件的多个不同 fd。“同时”指一个读写体操作完尚未 `close`，另一个读写体接着操作。

意义：多进程/多线程环境下让多个线程同时操作同一个大文件，减少读写时间、提升效率。

核心问题只有一个：**怎么造出多个指向同一文件的 fd**。

### 3.8.1 三种实现方式

**(1) 同一进程内多次 open 同一文件**：多个 fd → 多个文件表 → 同一个 inode。

![图3.8.1 同一进程多次 open 打开同一文件各数据结构关](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-20%20%E5%9B%BE3.8.1%20%E5%90%8C%E4%B8%80%E8%BF%9B%E7%A8%8B%E5%A4%9A%E6%AC%A1%20open%20%E6%89%93%E5%BC%80%E5%90%8C%E4%B8%80%E6%96%87%E4%BB%B6%E5%90%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%85%B3.png)

**(2) 不同进程分别 open 同一文件**：进程 1 的 fd1 → 进程 1 的文件表 1，进程 2 的 fd2 → 进程 2 的文件表 2，两张文件表都索引到同一 inode。

![图3.8.2 不同进程 open 打开同一文件数据结构关系](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-21%20%E5%9B%BE3.8.2%20%E4%B8%8D%E5%90%8C%E8%BF%9B%E7%A8%8B%20open%20%E6%89%93%E5%BC%80%E5%90%8C%E4%B8%80%E6%96%87%E4%BB%B6%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%85%B3%E7%B3%BB.png)

**(3) 同进程内 dup/dup2 复制 fd**：多个 fd 共享**同一个**文件表（偏移量、状态标志都共享）。

![图3.8.3 dup 复制文件描述符实现文件共享](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-22%20%E5%9B%BE3.8.3%20dup%20%E5%A4%8D%E5%88%B6%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E5%AE%9E%E7%8E%B0%E6%96%87%E4%BB%B6%E5%85%B1%E4%BA%AB.png)

⚫ 判断“分别写还是接续写”只需看一条：**是否共享同一文件表**。共享（dup）→ 接续写；不共享（多次 open）→ 分别写，除非加 `O_APPEND`。
⚫ 文件共享存在**竞争冒险**，见 3.9。

---

## 3.9 原子操作与竞争冒险

### 3.9.1 竞争冒险

Linux 是多任务、多进程系统，多个进程可能同时操作同一文件，该文件就是**共享资源**。裸机编程没有进程/多任务概念，Linux 下必须警惕竞争。

经典场景：进程 A、B 都对同一文件**追加写**，都没用 `O_APPEND`。A 运行，`lseek` 把偏移设为 1500（当时文件末尾），A 时间片耗尽；B 运行，`lseek` 也设为 1500；B `write` 100 字节，B 的偏移变 1600，B 时间片耗尽；A 恢复，从自己的偏移 **1500** 开始 `write` —— 而 1500 处已不是文件末尾，**A 的数据覆盖了 B 刚写的 100 字节**。

![图3.9.1 假设中的 AB 进程工作流程](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-23%20%E5%9B%BE3.9.1%20%E5%81%87%E8%AE%BE%E4%B8%AD%E7%9A%84%20AB%20%E8%BF%9B%E7%A8%8B%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

这种状态叫**竞争状态（竞争冒险）**：操作共享资源的进程/线程，结果**不可预期**，因为谁先拿到 CPU 完全由操作系统调度决定。

**原子操作**：由多步操作组成，要么一步不执行，一旦执行必须执行完所有步骤，不可能只执行其中子集。上例的病根是把“定位到末尾 + 写”拆成了两次独立调用。

### 3.9.2 用 O_APPEND 实现原子操作

`open` 带上 `O_APPEND` 后，每次 `write` 内部把“移动写偏移到文件末尾”和“写入数据”合成**一个原子操作**，所以绝不会出现 A 覆盖 B 的情况。这是 `O_APPEND` 最重要的价值——不只是“写末尾”这么简单。

### 3.9.3 pread 与 pwrite

与 `read`/`write` 功能相同，区别是自带 `offset` 参数，相当于 `lseek` + `read`/`write` 合体，并构成原子操作。

```c
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

| 参数 | 含义 |
|---|---|
| `fd`、`buf`、`count` | 与 `read`/`write` 完全相同 |
| `offset` | 本次读/写的位置偏移量 |

**返回值**：与 `read`/`write` 相同（实际读/写字节数，出错 -1）。

与 `lseek + read` 的两点区别：

⚫ 调用 `pread` 时**无法中断**其“定位 + 读”操作（即原子）；
⚫ **不更新文件表中记录的当前位置偏移量**。

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    unsigned char buffer[100];
    int fd;
    int ret;

    fd = open("./test_file", O_RDWR);
    if (-1 == fd) { perror("open error"); exit(-1); }

    /* 从偏移 1024 处读 100 字节 */
    ret = pread(fd, buffer, sizeof(buffer), 1024);
    if (-1 == ret) { perror("pread error"); goto err; }

    /* 查当前偏移量 */
    ret = lseek(fd, 0, SEEK_CUR);
    if (-1 == ret) { perror("lseek error"); goto err; }

    printf("Current Offset: %d\n", ret);
    ret = 0;
err:
    close(fd);
    exit(ret);
}
```

打印 `Current Offset: 0`（若换成 `read` 则打印 100）。

![图3.9.2 pread 函数测试结果 从](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-24%20%E5%9B%BE3.9.2%20pread%20%E5%87%BD%E6%95%B0%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E4%BB%8E.png)

⚫ `pwrite` 行为同理，同样不改变偏移量。

### 3.9.4 创建文件时的竞态

进程 A、B 都要打开一个**尚不存在**的文件。A 先 `open("./file", O_RDWR)` 失败；A 时间片耗尽；B 也失败，然后 B 用 `open("./file", O_RDWR | O_CREAT, ...)` 创建成功；B 时间片耗尽；A 恢复后用同样方式创建，**又成功**——同一个文件被创建两次，这是不允许的。

![图3.9.3 创建文件中存在的竞态](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-25%20%E5%9B%BE3.9.3%20%E5%88%9B%E5%BB%BA%E6%96%87%E4%BB%B6%E4%B8%AD%E5%AD%98%E5%9C%A8%E7%9A%84%E7%AB%9E%E6%80%81.png)

解法：`O_CREAT | O_EXCL` 同用。文件已存在则 `open` 返回错误，不存在才创建，把“判断是否存在 + 创建”合成**一个原子操作**，从而保证进程是文件的创建者。

---

## 3.10 fcntl 和 ioctl

### 3.10.1 fcntl

`fcntl()` 对**已打开**的 fd 执行一系列控制操作，相当于“多功能 fd 管理工具箱”：复制 fd（同 `dup`/`dup2`）、获取/设置文件描述符标志、获取/设置文件状态标志等。

```c
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* arg */);
```

| 参数 | 含义 |
|---|---|
| `fd` | 文件描述符 |
| `cmd` | 操作命令，均以 `F_XXX` 开头 |
| `...` | 可变参，第三个参数依 `cmd` 而定 |

`cmd` 五类功能：

| 功能 | cmd |
|---|---|
| 复制文件描述符 | `F_DUPFD`、`F_DUPFD_CLOEXEC` |
| 获取/设置**文件描述符标志** | `F_GETFD`、`F_SETFD` |
| 获取/设置**文件状态标志** | `F_GETFL`、`F_SETFL` |
| 获取/设置异步 IO 所有权 | `F_GETOWN`、`F_SETOWN` |
| 获取/设置记录锁 | `F_GETLK`、`F_SETLK` |

**返回值**：失败返回 -1 并设置 `errno`；成功时返回值**与 cmd 有关**——`F_DUPFD` 返回新 fd，`F_GETFD` 返回 fd 标志，`F_GETFL` 返回文件状态标志。

#### (1) 复制文件描述符

`cmd = F_DUPFD` 时，第三个参数表示“新 fd 必须 **≥ 该值** 且可用”；若该值已被占用，则取一个**大于**它的可用 fd。

```c
fd1 = open("./test_file", O_RDONLY);
if (-1 == fd1) { perror("open error"); exit(-1); }

fd2 = fcntl(fd1, F_DUPFD, 0);      /* 新 fd >= 0 的可用值 */
if (-1 == fd2) { perror("fcntl error"); ret = -1; goto err; }

printf("fd1: %d\nfd2: %d\n", fd1, fd2);   /* 这里 fd2 == 7，因为 0~6 已被占用 */
```

![图3.10.1 fcntl 复制文件描述符测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-26%20%E5%9B%BE3.10.1%20fcntl%20%E5%A4%8D%E5%88%B6%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

传 100 则 `fd2` 就是 100。

#### (2) 获取/设置文件状态标志

`F_GETFL` 不需第三参，返回状态标志；`F_SETFL` 需第三参（要设置的标志）。这些标志就是 `open` 的 `flags`。

⚫ **不能被 `F_SETFL` 修改的标志会被静默忽略**：文件权限标志（`O_RDONLY`/`O_WRONLY`/`O_RDWR`）与文件创建标志（`O_CREAT`/`O_EXCL`/`O_NOCTTY`/`O_TRUNC`）。
⚫ Linux 下**可修改**的只有：`O_APPEND`、`O_ASYNC`、`O_DIRECT`、`O_NOATIME`、`O_NONBLOCK`。

```c
int fd, ret, flag;

fd = open("./test_file", O_RDWR);
if (-1 == fd) { perror("open error"); exit(-1); }

flag = fcntl(fd, F_GETFL);                    /* 获取 */
if (-1 == flag) { perror("fcntl F_GETFL error"); ret = -1; goto err; }
printf("flags: 0x%x\n", flag);

ret = fcntl(fd, F_SETFL, flag | O_APPEND);    /* 追加 O_APPEND */
if (-1 == ret) { perror("fcntl F_SETFL error"); goto err; }

ret = 0;
err:
    close(fd);
    exit(ret);
```

![图3.10.2 fcntl 测试结果 2](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-27%20%E5%9B%BE3.10.2%20fcntl%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%202.png)

> 这就是“给一个已打开的 fd 动态加/去标志”的标准手法。

### 3.10.2 ioctl

`ioctl()` 是文件 IO 操作的“杂物箱”，处理的事情非常杂、不统一，主要用于**操作特殊文件或硬件外设**（如获取 LCD 信息），进阶篇会用到。

```c
#include <sys/ioctl.h>

int ioctl(int fd, unsigned long request, ...);
```

| 参数 | 含义 |
|---|---|
| `fd` | 文件描述符 |
| `request` | 请求的操作，与具体操作对象有关，**无统一值** |
| `...` | 可变参，依 `request` 决定 |

**返回值**：成功 0，失败 -1。

---

## 3.11 截断文件

```c
#include <unistd.h>
#include <sys/types.h>

int truncate(const char *path, off_t length);
int ftruncate(int fd, off_t length);
```

| 函数 | 指定方式 |
|---|---|
| `truncate` | 用**路径** `path` |
| `ftruncate` | 用**文件描述符** `fd` |

功能一样，把文件截断为 `length` 字节：

- 原大小 **>** `length`：多余数据丢失（被“砍”掉）；
- 原大小 **<** `length`：**扩展**，扩展部分读取得到空字节 `\0`。

⚫ `ftruncate` 前必须 `open` 拿到 fd，且必须具有**可写权限**（`O_WRONLY` 或 `O_RDWR`）。
⚫ **两个函数都不改变文件读写位置偏移量**。截断后若原偏移已越界，需重新 `lseek`，否则后续读写会出错。
⚫ 成功返回 0，失败 -1 并设置 `errno`。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(void)
{
    int fd;

    if (0 > (fd = open("./file1", O_RDWR))) {
        perror("open error");
        exit(-1);
    }

    /* 用 fd 把 file1 截断为 0 字节 */
    if (0 > ftruncate(fd, 0)) {
        perror("ftruncate error");
        exit(-1);
    }

    /* 用路径把 file2 截断为 1024 字节 */
    if (0 > truncate("./file2", 1024)) {
        perror("truncate error");
        exit(-1);
    }

    close(fd);
    exit(0);
}
```

测试前 file1、file2 均为 592 字节；运行后 file1 → 0 字节，file2 → 1024 字节。

![图3.11.1 准备 file1 文件和 file2 文件](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-28%20%E5%9B%BE3.11.1%20%E5%87%86%E5%A4%87%20file1%20%E6%96%87%E4%BB%B6%E5%92%8C%20file2%20%E6%96%87%E4%BB%B6.png)

![图3.11.2 测试结果](photo/%E7%AC%AC%E4%B8%89%E7%AB%A0%20%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6%E6%96%87%E4%BB%B6IO/3-29%20%E5%9B%BE3.11.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

---

## 本章速记

| 要点 | 结论 |
|---|---|
| 静态文件 | 磁盘上未被打开的文件；扇区 512B，块最常见 4KB，块是存取最小单位 |
| inode | 每文件唯一，记录大小/属主/权限/时间戳/类型/**数据 block 位置**；**不含文件名** |
| 打开三步 | 文件名 → inode 编号 → inode 结构体 → 按 block 位置读数据 |
| 动态文件 | `open` 时把数据读进内核缓冲区；读写都针对它，同步回磁盘由内核负责 |
| 三层结构 | 文件描述符表[fd] → **文件表**（状态标志/引用计数/**偏移量**/i-node 指针）→ inode |
| errno | `int` 全局变量，存**最近一次**错误号；**不是所有函数都设置**，需查 man |
| strerror | `char *strerror(int errnum)`，`<string.h>`，返回描述字符串 |
| perror | `void perror(const char *s)`，`<stdio.h>`，直接打印，自动加“`: `” |
| 退出方式 | `return` / `_exit()`、`_Exit()`（系统调用，**不刷新 stdio 缓冲**）/ `exit()`（库函数，会刷新）；推荐 `exit()` |
| 空洞文件 | `lseek` 越过文件尾再写，中间不占物理空间；`ls` 看逻辑大小，`du` 看实际占用 |
| O_TRUNC | 打开即清空为 0 字节，需可写打开 |
| O_APPEND | 每次 `write` 自动移到末尾；**不影响读**；**`lseek` 改不了写位置**；是**原子**的 |
| 多次 open 同一文件 | 多个 fd、多个文件表；**只有一份动态文件**；各 fd **偏移独立**；写是**分别写** |
| 引用计数 | 文件表记录，减到 0 才真正关闭动态文件 |
| dup | `int dup(int oldfd)`，系统分配最小可用 fd；与旧 fd **共享文件表**（共享偏移）→ 接续写 |
| dup2 | `int dup2(int oldfd, int newfd)`，手动指定新 fd；同样共享文件表 |
| 文件共享三法 | 同进程多次 open / 不同进程 open / dup(dup2) |
| 竞争冒险 | 多进程操作共享资源，调度顺序不可预期；病根是“定位 + 写”被拆成两步 |
| 原子操作 | 要么不执行、要么全执行；`O_APPEND`、`pread`/`pwrite`、`O_CREAT\|O_EXCL` 都是原子化手段 |
| pread/pwrite | 自带 `offset`，原子，**不改变文件表偏移量** |
| fcntl | `int fcntl(int fd, int cmd, ...)`；`F_DUPFD` 复制、`F_GETFL`/`F_SETFL` 读写状态标志 |
| F_SETFL 限制 | 只能改 `O_APPEND`/`O_ASYNC`/`O_DIRECT`/`O_NOATIME`/`O_NONBLOCK`，其余被忽略 |
| truncate/ftruncate | 按路径/按 fd 截断；变短砍掉、变长补 `\0`；**不改偏移量** |
