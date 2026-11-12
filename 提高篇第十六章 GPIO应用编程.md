# 第十六章 GPIO 应用编程

> 本章主题：`/sys/class/gpio` 目录结构与编号计算；GPIO 输出编程（gpio_out.c）；GPIO 输入编程（gpio_in.c）；GPIO 中断编程（poll + POLLPRI，gpio_intr.c）；开发板实测。

---

## 16.1 应用层如何操控 GPIO

GPIO 与 LED 一样，出厂系统里**没有 `/dev` 设备节点**，同样走 sysfs 方式操控，入口是 `/sys/class/gpio` 目录：

![图16.1.1 sysclassgpio 目录](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-1%20%E5%9B%BE16.1.1%20sysclassgpio%20%E7%9B%AE%E5%BD%95.png)

该目录下有两个文件、5 个文件夹：

| 名称 | 类型 | 作用 |
|---|---|---|
| `export` | 文件，**只写** | 将指定编号的 GPIO 引脚导出，导出成功后生成 `gpioX` 文件夹 |
| `unexport` | 文件，**只写** | 删除已导出的 GPIO 引脚，`gpioX` 文件夹随之消失 |
| `gpiochipX`（X=0/32/64/96/128） | 目录 | 5 个 GPIO **控制器**，对应 I.MX6UL/ULL 的 GPIO1~GPIO5，每个管理一组引脚 |

### 16.1.1 gpiochipX：GPIO 控制器的三个属性

进入任一 `gpiochipX` 目录（以 gpiochip0 为例）：

![图16.1.2 gpiochip0 目录下的文件](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-2%20%E5%9B%BE16.1.2%20gpiochip0%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6.png)

主要关注三个**只读**属性文件：

| 属性文件 | 含义 |
|---|---|
| `base` | 与 gpiochipX 的 X 相同——该组 GPIO 引脚的**最小编号**（GPIO 引脚靠编号操控） |
| `label` | 该组 GPIO 的标签（名字） |
| `ngpio` | 该控制器管理的引脚数量，所以编号范围是 **base ~ base+ngpio-1** |

![图16.1.3 base 属性文件](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-3%20%E5%9B%BE16.1.3%20base%20%E5%B1%9E%E6%80%A7%E6%96%87%E4%BB%B6.png)

![图16.1.4 label 属性文件](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-4%20%E5%9B%BE16.1.4%20label%20%E5%B1%9E%E6%80%A7%E6%96%87%E4%BB%B6.png)

![图16.1.5 ngpio 属性文件](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-5%20%E5%9B%BE16.1.5%20ngpio%20%E5%B1%9E%E6%80%A7%E6%96%87%E4%BB%B6.png)

### 16.1.2 GPIO 编号计算（本章最核心的换算）

芯片手册里的引脚名（如 `GPIO4_IO16`）和 sysfs 编号的换算规则：**编号 = 所在控制器的 base + IO 序号**。

| 芯片手册引脚名 | 控制器 | 计算 | sysfs 编号 |
|---|---|---|---|
| GPIO4_IO16 | GPIO4 → gpiochip96 | 96 + 16 | **112** |
| GPIO3_IO20 | GPIO3 → gpiochip64 | 64 + 20 | **84** |
| GPIO1_IO0 | GPIO1 → gpiochip0 | 0 + 0 | 0 |

（GPIO1~GPIO5 依次对应 gpiochip0/32/64/96/128，每组 32 个引脚。）

### 16.1.3 export 与 unexport

使用引脚前必须先导出（往 `export` 写入编号字符串）：

```bash
echo 0 > export      # 导出编号 0 的引脚（I.MX6ULL 上即 GPIO1_IO00）
```

导出成功后 `/sys/class/gpio` 下会生成 `gpio0` 文件夹：

![图16.1.6 导出成功](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-6%20%E5%9B%BE16.1.6%20%E5%AF%BC%E5%87%BA%E6%88%90%E5%8A%9F.png)

![图16.1.7 GPIO 引脚对应的文件夹](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-7%20%E5%9B%BE16.1.7%20GPIO%20%E5%BC%95%E8%84%9A%E5%AF%B9%E5%BA%94%E7%9A%84%E6%96%87%E4%BB%B6%E5%A4%B9.png)

用完后删除：

```bash
echo 0 > unexport    # 删除导出的编号 0 引脚，gpio0 文件夹消失
```

⚫ **不是所有引脚都能导出**：如果该引脚已经在内核中被某个驱动占用，导出会失败：

![图16.1.8 导出失败](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-8%20%E5%9B%BE16.1.8%20%E5%AF%BC%E5%87%BA%E5%A4%B1%E8%B4%A5.png)

### 16.1.4 gpioX 目录的四个属性文件

导出生成的 `gpioX` 文件夹：

![图16.1.9 gpioX 文件夹](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-9%20%E5%9B%BE16.1.9%20gpioX%20%E6%96%87%E4%BB%B6%E5%A4%B9.png)

重点关心其中四个属性文件：

**① direction —— 输入/输出模式**（可读可写，值只能 `"out"` 或 `"in"`）

```bash
echo "out" > direction    # 输出模式
echo "in"  > direction    # 输入模式
cat direction             # 查看当前模式
```

![图16.1.10 direction 属性文件](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-10%20%E5%9B%BE16.1.10%20direction%20%E5%B1%9E%E6%80%A7%E6%96%87%E4%BB%B6.png)

**② value —— 电平状态**（可读可写）

```bash
# 输出模式：写 0/1 控制输出低/高电平
echo "out" > direction
echo "1"   > value        # 输出高电平
echo "0"   > value        # 输出低电平

# 输入模式：读 value 获取引脚输入电平
echo "in"  > direction
cat value
```

⚫ **只有输入模式下才能 `cat value` 读电平**；输出模式下读不到，只能外接 LED 或用万用表测。

**③ active_low —— 极性反转**（可读可写，默认 0）

```bash
# active_low = 0（正常极性）
echo "1" > value    # 输出高
echo "0" > value    # 输出低

# active_low = 1（极性反转）
echo "1" > value    # 输出低！
echo "0" > value    # 输出高！
```

对输入模式同样生效。

**④ edge —— 中断触发模式**（可读可写；配置前必须先设为输入模式）

```bash
echo "none"    > edge    # 非中断引脚
echo "rising"  > edge    # 上升沿触发
echo "falling" > edge    # 下降沿触发
echo "both"    > edge    # 边沿触发（上升沿+下降沿）
```

引脚配置为中断后，可以用 **poll() 监听** value 文件描述符来等待电平变化（见 16.4）。

---

## 16.2 GPIO 应用编程之输出

例程源码：光盘 → `11、Linux C 应用编程例程源码` → `16_gpio` → `gpio_out.c`。

> **文件名说明**（同第 15 章）：原书演示截图里产物叫 `testApp`，本笔记按例程文件名统一叫 `gpio_out`，看图时按别名理解。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

static char gpio_path[100];

/* 通用：向 gpioX 目录下的 attr 属性文件写入字符串 val */
static int gpio_config(const char *attr, const char *val)
{
    char file_path[100];
    int len;
    int fd;

    sprintf(file_path, "%s/%s", gpio_path, attr);
    if (0 > (fd = open(file_path, O_WRONLY))) {
        perror("open error");
        return fd;
    }

    len = strlen(val);
    if (len != write(fd, val, len)) {
        perror("write error");
        close(fd);
        return -1;
    }

    close(fd);
    return 0;
}

int main(int argc, char *argv[])
{
    /* 校验传参：<gpio 编号> <输出电平> */
    if (3 != argc) {
        fprintf(stderr, "usage: %s <gpio> <value>\n", argv[0]);
        exit(-1);
    }

    /* 判断指定编号的 GPIO 是否已导出（gpioX 目录是否存在） */
    sprintf(gpio_path, "/sys/class/gpio/gpio%s", argv[1]);

    if (access(gpio_path, F_OK)) {          // 目录不存在 → 需要导出
        int fd;
        int len;

        if (0 > (fd = open("/sys/class/gpio/export", O_WRONLY))) {
            perror("open error");
            exit(-1);
        }

        len = strlen(argv[1]);
        if (len != write(fd, argv[1], len)) {   // 导出 gpio
            perror("write error");
            close(fd);
            exit(-1);
        }

        close(fd);
    }

    /* 配置为输出模式 */
    if (gpio_config("direction", "out"))
        exit(-1);

    /* 极性设置（不配也可以） */
    if (gpio_config("active_low", "0"))
        exit(-1);

    /* 控制输出电平：argv[2] 为 "1" 或 "0" */
    if (gpio_config("value", argv[2]))
        exit(-1);

    exit(0);
}
```

用法与逻辑：

```bash
./gpio_out <编号> <0|1>     # 0=低电平，1=高电平
```

流程：`access()` 判断 gpioX 目录是否存在 → 不存在则写 `export` 导出 → `direction` 写 `"out"` → `active_low` 写 `"0"` → `value` 写目标电平。封装的 `gpio_config()` 是后面三个例程共用的模板函数。

编译（交叉编译环境见第 15 章 15.3.1）：

![图16.2.1 编译应用程序](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-11%20%E5%9B%BE16.2.1%20%E7%BC%96%E8%AF%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

---

## 16.3 GPIO 应用编程之输入

例程源码：`16_gpio` → `gpio_in.c`。读取指定引脚的输入电平并打印。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

static char gpio_path[100];

/* gpio_config() 与 gpio_out.c 完全相同，略 */
static int gpio_config(const char *attr, const char *val)
{
    char file_path[100];
    int len;
    int fd;

    sprintf(file_path, "%s/%s", gpio_path, attr);
    if (0 > (fd = open(file_path, O_WRONLY))) {
        perror("open error");
        return fd;
    }

    len = strlen(val);
    if (len != write(fd, val, len)) {
        perror("write error");
        close(fd);
        return -1;
    }

    close(fd);
    return 0;
}

int main(int argc, char *argv[])
{
    char file_path[100];
    char val;
    int fd;

    /* 校验传参：<gpio 编号> */
    if (2 != argc) {
        fprintf(stderr, "usage: %s <gpio>\n", argv[0]);
        exit(-1);
    }

    /* 判断是否已导出，未导出则导出 */
    sprintf(gpio_path, "/sys/class/gpio/gpio%s", argv[1]);

    if (access(gpio_path, F_OK)) {
        int len;

        if (0 > (fd = open("/sys/class/gpio/export", O_WRONLY))) {
            perror("open error");
            exit(-1);
        }

        len = strlen(argv[1]);
        if (len != write(fd, argv[1], len)) {
            perror("write error");
            close(fd);
            exit(-1);
        }

        close(fd);
    }

    /* 配置为输入模式 */
    if (gpio_config("direction", "in"))
        exit(-1);

    /* 极性设置 */
    if (gpio_config("active_low", "0"))
        exit(-1);

    /* 配置为非中断方式 */
    if (gpio_config("edge", "none"))
        exit(-1);

    /* 打开 value 属性文件读取电平 */
    sprintf(file_path, "%s/%s", gpio_path, "value");

    if (0 > (fd = open(file_path, O_RDONLY))) {
        perror("open error");
        exit(-1);
    }

    if (0 > read(fd, &val, 1)) {        // 读 1 个字节即可："0" 或 "1"
        perror("read error");
        close(fd);
        exit(-1);
    }

    printf("value: %c\n", val);

    close(fd);
    exit(0);
}
```

与输出程序的差别：`direction` 写 `"in"`、额外把 `edge` 写 `"none"`（非中断）、最后以 **`O_RDONLY` 打开 value 读 1 字节**打印。用法：

```bash
./gpio_in <编号>
```

![图16.3.1 编译示例代码](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-12%20%E5%9B%BE16.3.1%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

---

## 16.4 GPIO 应用编程之中断

例程源码：`16_gpio` → `gpio_intr.c`。把引脚配成**边沿触发**，用 poll() 等待中断并打印。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <poll.h>           // poll() 需要的头文件

static char gpio_path[100];

/* gpio_config() 同前，略 */
static int gpio_config(const char *attr, const char *val)
{
    char file_path[100];
    int len;
    int fd;

    sprintf(file_path, "%s/%s", gpio_path, attr);
    if (0 > (fd = open(file_path, O_WRONLY))) {
        perror("open error");
        return fd;
    }

    len = strlen(val);
    if (len != write(fd, val, len)) {
        perror("write error");
        return -1;
    }

    close(fd);
    return 0;
}

int main(int argc, char *argv[])
{
    struct pollfd pfd;
    char file_path[100];
    int ret;
    char val;

    /* 校验传参 */
    if (2 != argc) {
        fprintf(stderr, "usage: %s <gpio>\n", argv[0]);
        exit(-1);
    }

    /* 判断是否已导出，未导出则导出 */
    sprintf(gpio_path, "/sys/class/gpio/gpio%s", argv[1]);

    if (access(gpio_path, F_OK)) {
        int len;
        int fd;

        if (0 > (fd = open("/sys/class/gpio/export", O_WRONLY))) {
            perror("open error");
            exit(-1);
        }

        len = strlen(argv[1]);
        if (len != write(fd, argv[1], len)) {
            perror("write error");
            exit(-1);
        }

        close(fd);
    }

    /* 配置为输入模式 */
    if (gpio_config("direction", "in"))
        exit(-1);

    /* 极性设置 */
    if (gpio_config("active_low", "0"))
        exit(-1);

    /* 中断触发方式：上升沿 + 下降沿（边沿触发） */
    if (gpio_config("edge", "both"))
        exit(-1);

    /* 打开 value 属性文件 */
    sprintf(file_path, "%s/%s", gpio_path, "value");

    if (0 > (pfd.fd = open(file_path, O_RDONLY))) {
        perror("open error");
        exit(-1);
    }

    /* 只关心高优先级数据可读（中断） */
    pfd.events = POLLPRI;

    read(pfd.fd, &val, 1);          // 先读一次清除当前状态
    for ( ; ; ) {

        ret = poll(&pfd, 1, -1);    // 阻塞等待，-1 = 永不超时
        if (0 > ret) {
            perror("poll error");
            exit(-1);
        } else if (0 == ret) {
            fprintf(stderr, "poll timeout.\n");
            continue;
        }

        /* 校验高优先级数据是否可读 */
        if (pfd.revents & POLLPRI) {
            if (0 > lseek(pfd.fd, 0, SEEK_SET)) {   // 读位置移回头部
                perror("lseek error");
                exit(-1);
            }

            if (0 > read(pfd.fd, &val, 1)) {
                perror("read error");
                exit(-1);
            }

            printf("GPIO 中断触发<value=%c>\n", val);
        }
    }

    exit(0);
}
```

**为什么必须用 poll() 而不是直接阻塞 read()？**

poll() 可以监视一个或多个文件描述符的 I/O 状态变化：`POLLIN`/`POLLOUT` 是**普通优先级**数据可读/可写，`POLLPRI` 是**高优先级**数据可读——**中断就是一种高优先级事件**。中断触发时 value 变为"高优先级可读"，poll() 返回，`revents & POLLPRI` 成立。

⚫ 三个关键细节（漏掉任何一个程序都不工作）：

| 细节 | 原因 |
|---|---|
| 进 for 循环前**先 read 一次** | 清除当前状态，否则残留状态会让 poll 立即返回 |
| poll 返回后 **lseek 回到文件头** 再 read | value 是 sysfs 属性文件，读完位置在末尾，不 lseek 会一直读到 EOF |
| 事件判断用 **`revents & POLLPRI`** | 而不是 POLLIN——中断是高优先级事件 |

除了 poll()，还可以用第 13 章 13.4 节的**异步 I/O（SIGIO）**方式监视 GPIO 中断。

![图16.4.1 编译应用程序](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-13%20%E5%9B%BE16.4.1%20%E7%BC%96%E8%AF%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

---

## 16.5 在开发板上测试

三个程序对应的产物（原书叫 `testApp`，本笔记分别对应 `gpio_out`/`gpio_in`/`gpio_intr`）都要拷到板子上执行。测试引脚选**底板上已引出的 GPIO1_IO01**（ALPHA 板丝印 GPIO1；Mini 板通过背面丝印或原理图确认）：

![图16.5.1 GPIO1_IO01引脚 ALPHA板](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-14%20%E5%9B%BE16.5.1%20GPIO1_IO01%E5%BC%95%E8%84%9A%20ALPHA%E6%9D%BF.jpg)

![图16.5.1 GPIO1_IO01引脚 Mini板](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-15%20%E5%9B%BE16.5.1%20GPIO1_IO01%E5%BC%95%E8%84%9A%20Mini%E6%9D%BF.jpg)

GPIO1_IO01 的编号按 16.1.2 换算：GPIO1 → base 0，IO01 → **编号 1**。

### GPIO 输出测试

```bash
./gpio_out 1 1     # GPIO1_IO01 输出高电平
./gpio_out 1 0     # 输出低电平
```

![图16.5.2 控制GPIO输出的电平状态](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-16%20%E5%9B%BE16.5.2%20%E6%8E%A7%E5%88%B6GPIO%E8%BE%93%E5%87%BA%E7%9A%84%E7%94%B5%E5%B9%B3%E7%8A%B6%E6%80%81.png)

执行后用万用表量引脚电压、或接一个 LED 小灯验证。

### GPIO 输入测试

先用**杜邦线**把 GPIO1_IO01 接到板上 **3.3V 电源引脚**，读高电平：

![图16.5.3 高电平读取](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-17%20%E5%9B%BE16.5.3%20%E9%AB%98%E7%94%B5%E5%B9%B3%E8%AF%BB%E5%8F%96.png)

打印 `value: 1`，确实是高电平；再改接到 **GND** 引脚：

![图16.5.4 低电平读取](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-19%20%E5%9B%BE16.5.4%20%E4%BD%8E%E7%94%B5%E5%B9%B3%E8%AF%BB%E5%8F%96.png)

打印 `value: 0`，与实际相符。

### GPIO 中断测试

```bash
./gpio_intr 1      # 监测 GPIO1_IO01 中断触发
```

![图16.5.5 中断触发测试结果](photo/%E7%AC%AC%E5%8D%81%E5%85%AD%E7%AB%A0%20GPIO%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/16-18%20%E5%9B%BE16.5.5%20%E4%B8%AD%E6%96%AD%E8%A7%A6%E5%8F%91%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

程序跑起来后，用杜邦线把 GPIO1_IO01 在 **GND 和 3.3V 之间来回切换**，制造由高到低、由低到高的电平变化；每切换一次，poll() 被唤醒并打印一次 `GPIO 中断触发<value=x>`。测试完 **Ctrl+C 退出**（程序是死循环）。

---

## 16.6 编译与部署

交叉编译环境与第 15 章完全相同（工具链已在 PATH 里），三份源码用同一份 Makefile，改 `TARGET`/`SRCS` 即可；或者最直接的一条命令：

```bash
cd ~/linux/IMX6ULL/application/16_gpio
arm-linux-gnueabihf-gcc gpio_out.c -o gpio_out
arm-linux-gnueabihf-gcc gpio_in.c  -o gpio_in
arm-linux-gnueabihf-gcc gpio_intr.c -o gpio_intr
file gpio_out        # 确认 ARM 32 位
```

传板子（**目标必须带完整文件名**，Dropbear 的坑见 15.4）：

```bash
scp gpio_out gpio_in gpio_intr root@192.168.1.104:/root/
```

板上运行前先 `chmod +x gpio_*`。测试完成后建议 `echo 1 > /sys/class/gpio/unexport` 把用过的引脚删除，保持环境干净。

---

## 本章速记

| 要点 | 结论 |
|---|---|
| GPIO 操控方式 | sysfs，入口 `/sys/class/gpio`，无 `/dev` 节点 |
| 5 个控制器 | GPIO1~GPIO5 ↔ gpiochip0/32/64/96/128（各 32 脚） |
| 编号换算 | **编号 = base + IO 序号**，如 GPIO4_IO16 = 96+16 = 112 |
| 导出/删除 | `echo N > export` 生成 gpioN；`echo N > unexport` 删除；都是只写文件 |
| 导出失败 | 引脚已被内核驱动占用 |
| gpioX 四属性 | `direction`(in/out)、`value`(0/1)、`active_low`(极性)、`edge`(none/rising/falling/both) |
| 输入读电平 | **只有输入模式才能 `cat value`**；输出模式读不到 |
| active_low=1 | 读写电平全部反转，输入输出都受影响 |
| edge 配置前提 | 先把 direction 设为 in |
| 中断等待 | poll() + **POLLPRI**（高优先级事件），不能用阻塞 read |
| 中断三细节 | 先读一次清状态；poll 返回后 lseek(0, SEEK_SET) 再读；判断 `revents & POLLPRI` |
| 传输板子 | `scp 产物 root@IP:/root/led` 形式——**目标必须带文件名** |
