# 第十五章 点亮 LED

> 提高篇第一章。从本章开始不再是在 Ubuntu 上"自娱自乐"，而是真正操控开发板硬件。Linux 下**一切皆文件**，硬件也以文件形式呈现，所以本篇没有新的编程知识点——用的全是入门篇的 `open/read/write/close`，重点在于：**针对不同硬件，去找它对应的那个"文件"**。
>
> 本章主题：应用层操控硬件的两种方式（设备节点 vs sysfs 属性文件）；sysfs 文件系统与 `/sys` 目录结构；LED 的三个属性文件 `brightness`/`max_brightness`/`trigger`；编写 LED 应用程序并交叉编译到开发板测试。

ALPHA/Mini I.MX6U 开发板（核心板 + 底板）上共有 3 颗 LED，只有 **1 颗**可供用户控制，另外两颗是电源指示灯。板上唯一可控的这颗 LED 由 GPIO 控制。

---

## 15.1 应用层操控硬件的两种方式

Linux 下一切皆文件，应用层操控底层硬件同样通过文件 I/O 实现。**设备文件**（字符设备/块设备文件）就是硬件设备向应用层提供的接口，LCD、串口、按键、摄像头等都与设备文件一一对应。设备文件通常位于 `/dev/` 目录下，所以也叫**设备节点**。

但设备节点**不是**操控硬件的唯一途径——另一种方式是 **sysfs 文件系统**。

### 15.1.1 sysfs 文件系统

sysfs 是一个**基于内存的虚拟文件系统**（同 devfs、proc 一样），作用是把内核信息以文件的方式提供给应用层：把连接在系统上的设备和总线组织成分级的文件结构，展示设备驱动模型中各组件的层次关系，将内核对象（kernel object，譬如一个硬件设备）的数据、属性导出到用户空间。

内核组成要素在 sysfs 中的表现（表 15.1.1）：

| 内核中的组成要素 | sysfs 中的表现 |
|---|---|
| 内核对象（譬如一个硬件设备） | 目录 |
| 对象属性（譬如设备属性） | 文件 |
| 对象关系 | 链接文件 |

![图15.1.1 sys目录](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-1%20%E5%9B%BE15.1.1%20sys%20%E7%9B%AE%E5%BD%95.png)

### 15.1.2 sysfs 与 /sys

sysfs 挂载在 `/sys` 目录下。开发板出厂系统进入 `/sys` 后可见 `block、bus、class、dev、devices、firmware、fs、kernel、modules、power` 等子目录（上图），各自的含义（表 15.1.2）：

| /sys 下的子目录 | 说明 |
|---|---|
| `/sys/devices` | 系统中**所有设备**的存放目录，是 sysfs 管理设备的**最重要**目录结构 |
| `/sys/block` | 块设备目录（过时的接口），其中的文件通常是链接到 `/sys/devices` |
| `/sys/bus` | 按**总线类型**分类放置设备（如 i2c 设备挂在 I2C 总线下），文件链接到 `/sys/devices` |
| `/sys/class` | 按**功能分类**放置设备（如 `/sys/class/leds` 存放所有 LED 设备、`/sys/class/input` 存放输入类设备），文件链接到 `/sys/devices` |
| `/sys/dev` | 按**设备号**（major:minor）方式放置，链接文件链接到 `/sys/devices` |
| `/sys/firmware` | 描述内核中的固件 |
| `/sys/fs` | 描述系统中所有文件系统及已挂载点 |
| `/sys/kernel` | 内核中所有可调参数的位置 |
| `/sys/module` | 系统中所有模块的信息 |
| `/sys/power` | 系统电源选项，可控制整个系统的电源状态 |

记忆主线：**所有设备都在 `/sys/devices` 体现；`/sys/bus`、`/sys/class`、`/sys/dev` 只是按总线类型、功能分类、设备号三种视角组织的链接**。

设备的数据、属性通过设备目录下的**属性文件**导出：读属性文件 = 读取设备属性信息；写属性文件 = 设置设备属性、控制设备状态。

### 15.1.3 总结：两种操控方式

- ⚫ `/dev/` 目录下的设备文件（设备节点）；
- ⚫ `/sys/` 目录下设备的属性文件。

**如何选择**（取决于驱动实现）：

- 简单设备（LED、GPIO 等）：驱动一般基于标准驱动框架实现，把属性导出到 sysfs，**用 sysfs 属性文件操控**；
- 复杂设备（LCD、触摸屏、摄像头等）：通常**只有设备节点**，通过 `ioctl` 等方式操控。

### 15.1.4 标准接口与非标准接口

Linux 内核为降低驱动开发难度、统一接口，提出了**设备驱动框架**概念：针对常见设备分类（LED 类、输入类、FrameBuffer 类、video 类、PWM 类等），为每类设备设计一套成熟的、标准的驱动实现框架。基于框架写的驱动，向上层提供**统一、标准化的接口**——这就是"标准接口"。

不使用内核框架也可以写出驱动，但这样注册的设备就不是标准设备，接口是非标准的。此外，很多外设无法归入内核规划的任何设备类型（Linux 为此设了**杂散/杂项类设备 misc device** 兜底），杂项设备驱动提供的接口通常就是**非标准接口**——具体怎么操控往往只有驱动工程师自己知道。嵌入式系统中很多外设驱动都是定制的，应用层拿到一块新板子时，第一件事就是搞清楚目标设备走的是哪种接口。

---

## 15.2 LED 硬件控制方式

ALPHA/Mini I.MX6U 出厂系统中，用户 LED 使用**内核标准 LED 驱动框架**注册，`/dev` 下**没有**对应设备节点，走 **sysfs 方式**控制。

![图15.2.1 用户LED灯 ALPHA板](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-2%20%E5%9B%BE15.2.1%20%E7%94%A8%E6%88%B7LED%E7%81%AF%20ALPHA%E6%9D%BF.jpg)

![图15.2.1 用户LED灯 Mini板](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-3%20%E5%9B%BE15.2.1%20%E7%94%A8%E6%88%B7LED%E7%81%AF%20Mini%E6%9D%BF.png)

板上箭头所指即用户 LED（另一颗 PWR 是电源指示灯）。LED 设备目录在 `/sys/class/leds` 下：

![图15.2.2 sysclassleds目录](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-4%20%E5%9B%BE15.2.2%20sysclassleds%E7%9B%AE%E5%BD%95.png)

目录下有 `beep`、`mmc0::`、`sys-led` 等；`sys-led` 就是底板用户 LED 的设备文件夹：

![图15.2.3 sys-led文件夹](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-5%20%E5%9B%BE15.2.3%20sys-led%E6%96%87%E4%BB%B6%E5%A4%B9.png)

重点关注的三个**属性文件**：

| 属性文件 | 权限 | 作用 |
|---|---|---|
| `brightness` | 可读可写 | 设置/获取 LED 亮度等级。**0 = 灭，非 0 正整数 = 亮**，值越大越亮。PWM 控制的 LED 才有真正的多级亮度（不同等级对应不同占空比）；GPIO 控制的 LED 只有亮/灭两种状态 |
| `max_brightness` | 只读 | 获取 LED 设备的最大亮度等级 |
| `trigger` | 可读可写 | 触发模式。读 = 获取当前触发模式；写 = 设置触发模式。LED 会根据不同触发条件自动控制亮灭 |

用 `cat trigger` 可查看 LED 支持的所有触发模式，**方括号括起来的是当前生效模式**：

![图15.2.4 LED支持的触发模式](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-6%20%E5%9B%BE15.2.4%20LED%E6%94%AF%E6%8C%81%E7%9A%84%E8%A7%A6%E5%8F%91%E6%A8%A1%E5%BC%8F.png)

常用触发模式：

| 模式 | 行为 |
|---|---|
| `none` | 无触发（手动控制） |
| `mmc0` | 对 mmc0 设备发起读写时 LED 闪烁 |
| `timer` | LED 有规律地一亮一灭（定时器控制） |
| `heartbeat` | 心跳呼吸模式，模仿人心跳的亮灭变化 |

系统启动后通常会把一颗 LED 设为 `heartbeat` 作为系统正常运行指示灯——ALPHA/Mini 板出厂即是如此。

命令行直接控制（写入的都是**字符串**）：

```bash
echo timer > trigger    # 触发模式设为 timer
echo none > trigger     # 取消触发模式
echo 1 > brightness     # 点亮 LED
echo 0 > brightness     # 熄灭 LED
```

Tips：`cat` 读到的、`echo` 写入的都是字符串；所以应用程序里用 `write()` 写属性文件同样要写字符串，`read()` 读出来的也是 ASCII 字符串。

---

## 15.3 编写 LED 应用程序

例程源码路径：开发板光盘 → `11、Linux C 应用编程例程源码` → `15_led` → `led.c`。

> **文件名说明（先看这里，避免看图困惑）**：原书的演示截图里，源码叫 `testApp.c`、编译产物叫 `testApp`（下图 15.3.1 里就是 `${CC} -o testApp testApp.c`）——那是作者写书演示时改的名。**光盘例程和本笔记统一用 `led.c` → `led`**，两者是同一个程序，只是名字不同。后文插图里出现的 `testApp` 一律按 `led` 理解即可。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define LED_TRIGGER    "/sys/class/leds/sys-led/trigger"
#define LED_BRIGHTNESS "/sys/class/leds/sys-led/brightness"
#define USAGE() fprintf(stderr, "usage:\n"           \
        "  %s <on|off>\n"                            \
        "  %s <trigger> <type>\n", argv[0], argv[0])

int main(int argc, char *argv[])
{
    int fd1, fd2;

    /* 校验传参 */
    if (2 > argc) {
        USAGE();
        exit(-1);
    }

    /* 打开属性文件 */
    fd1 = open(LED_TRIGGER, O_RDWR);
    if (0 > fd1) {
        perror("open error");
        exit(-1);
    }

    fd2 = open(LED_BRIGHTNESS, O_RDWR);
    if (0 > fd2) {
        perror("open error");
        exit(-1);
    }

    /* 根据传参控制 LED */
    if (!strcmp(argv[1], "on")) {
        write(fd1, "none", 4);      // 先将触发模式设置为 none
        write(fd2, "1", 1);         // 点亮 LED
    } else if (!strcmp(argv[1], "off")) {
        write(fd1, "none", 4);      // 先将触发模式设置为 none
        write(fd2, "0", 1);         // LED 灭
    } else if (!strcmp(argv[1], "trigger")) {
        if (3 != argc) {
            USAGE();
            exit(-1);
        }
        if (0 > write(fd1, argv[2], strlen(argv[2])))
            perror("write error");
    } else
        USAGE();

    exit(0);
}
```

程序逻辑：`open()` 打开 `trigger` 和 `brightness` 两个属性文件 → 按参数分发：`on`/`off` 都是**先把 `"none"` 写入 trigger 解除自动触发**，再向 brightness 写 `"1"`/`"0"`（否则 heartbeat 等触发模式会盖掉手动设置）；`trigger <type>` 则把第二个参数原样写入 trigger 文件。写入的都是字符串，与 15.2 的 Tips 一致。

### 15.3.1 交叉编译工具链的安装与验证

**为什么要交叉编译**：开发板是 ARM 架构（i.MX6ULL = Cortex-A7），Ubuntu 是 x86_64。Ubuntu 自带的 `gcc` 只能生成 x86 程序，放到板子上根本跑不起来（报 `cannot execute binary file`）。必须用**运行在 x86 上、生成 ARM 代码**的编译器——这就是交叉编译器（cross compiler）。

正点原子提供的是 Linaro 版：`gcc-linaro-4.9.4-2017.01`，目标三元组 `arm-linux-gnueabihf`（arm 架构 / linux 系统 / gnueabihf 硬浮点 ABI）。

#### ① 选对安装包

光盘 `05、开发工具/01、交叉编译器/` 下有两个包，**按主机位数选**：

| 安装包 | 适用主机 | 说明 |
|---|---|---|
| `gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf.tar.xz` | **64 位 Ubuntu** | 现在绝大多数虚拟机都是这个，**优先选它** |
| `gcc-linaro-4.9.4-2017.01-i686_arm-linux-gnueabihf.tar.xz` | 32 位 Ubuntu | 64 位系统用它会缺 32 位运行库，还要额外装 `lib32stdc++6` 等 |

用 `uname -m` 判断：输出 `x86_64` 就用 x86_64 包，输出 `i686`/`i386` 才用 i686 包。

#### ② 解压安装

```bash
sudo mkdir -p /usr/local/arm
sudo tar -vxf gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf.tar.xz -C /usr/local/arm/
```

包内顶层目录名就是 `gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf`，解压后完整路径：

```
/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/
```

（放 `/usr/local/arm` 是正点原子的约定，放别处也行，后面的 PATH 跟着改。）

#### ③ 加入 PATH

打开 `~/.bashrc`，在末尾加一行：

```bash
export PATH=$PATH:/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin
```

保存后让配置立即生效（或者干脆**新开一个终端**）：

```bash
source ~/.bashrc
```

> **注意**：必须写 `$PATH:` 前缀。直接写 `export PATH=/usr/local/arm/.../bin` 会把系统原有路径全部清掉，`ls`、`vim` 全都找不到了。

#### ④ 验证安装成功（三条命令）

```bash
arm-linux-gnueabihf-gcc -v          # 能否找到 + 版本
which arm-linux-gnueabihf-gcc       # 实际用的是哪个
arm-linux-gnueabihf-gcc --version   # 简版
```

通过判据：

| 检查项 | 期望输出 |
|---|---|
| `-v` 最后一行 | `gcc version 4.9.4 (Linaro GCC 4.9-2017.01)` |
| `which` | `/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc` |
| `--version` | `arm-linux-gnueabihf-gcc (Linaro GCC 4.9-2017.01) 4.9.4` |

`-v` 输出的 `Configured with` 里能看到三个关键参数，正好对应 i.MX6ULL：

```
--with-arch=armv7-a           Cortex-A7 指令集
--with-float=hard             硬浮点
--with-fpu=vfpv3-d16          VFPv3-D16 浮点单元
```

若报 `command not found`：`echo $PATH` 看末尾有没有那个 `bin` 目录——没有就是 `.bashrc` 没生效（改完要 `source` 或重开终端）。

> 小提示：`-v` 显示 `gcc version 4.9.4`，而 `--version` 显示 `(Linaro GCC 4.9-2017.01) 4.9.4`——两种格式，**是同一个编译器**，不要以为是装了两个。

#### ⑤ 两种编译方式，别混用

| 方式 | 编译器 | 适用 | 说明 |
|---|---|---|---|
| **独立工具链**（推荐） | `arm-linux-gnueabihf-gcc` | 本书所有应用例程 | 上面装的这套，命令短、不污染环境 |
| **Yocto SDK**（原书用的） | `${CC}` = `arm-poky-linux-gnueabi-gcc` | 编译 uboot、内核 | 需先 `source` 环境脚本，带一长串 `--sysroot` 预设参数 |

两者 ABI 都是 armhf，但 **sysroot 和 glibc 版本不同**。如果先 `source` 了 SDK 再去敲 `arm-linux-gnueabihf-gcc`，可能出现头文件/库混用导致的诡异错误。**一个终端里只用一种**，要切换就关掉终端重开。

### 15.3.2 原书用的 Yocto SDK 方式

在开发板上运行，必须用 I.MX6U 平台的交叉编译工具（安装方法见《【正点原子】I.MX6U 用户快速体验 V1.7.3》第四章）。使用前先 `source` 环境设置脚本：

```bash
source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
```

（`/opt/fsl-imx-x11/4.1.15-2.1.0` 是工具链安装目录，按自己的实际路径来。）

![图15.3.1 交叉编译应用程序](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-7%20%E5%9B%BE15.3.1%20%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

（注意截图里源码是 `testApp.c`、产物是 `testApp`——按 15.3 开头的说明，对应本笔记的 `led.c` → `led`。）

编译直接用 `${CC}` 变量，它就是交叉编译器 `arm-poky-linux-gnueabi-gcc`（带一串预设选项）：

![图15.3.2 变量 CC](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-8%20%E5%9B%BE15.3.2%20%E5%8F%98%E9%87%8F%20CC.png)

编译生成 `testApp`（对应本笔记的 `led`），用 `file` 确认是 32 位 ARM 可执行文件：

![图15.3.3 testApp 文件类型](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-9%20%E5%9B%BE15.3.3%20testApp%20%E6%96%87%E4%BB%B6%E7%B1%BB%E5%9E%8B.png)

### 15.3.3 用独立工具链编译本章程序（实际步骤）

本工程在 Ubuntu 里的路径按正点原子例程目录组织：

```bash
cd ~/linux/IMX6ULL/application/15_led
ls          # led.c
```

**最直接的一条命令**：

```bash
arm-linux-gnueabihf-gcc led.c -o led
```

**验证产物**（四条，逐层确认）：

```bash
file led
# 期望：ELF 32-bit LSB executable, ARM, EABI5 ... interpreter /lib/ld-linux-armhf.so.3

readelf -h led | grep -E "Class|Machine"
# 期望：ELF32 / ARM

readelf -d led | grep NEEDED
# 期望：libc.so.6

file led | grep -q "ARM" && echo "OK: 是 ARM 程序" || echo "错误: 不是 ARM"
```

三个特征缺一不可：

| 特征 | 含义 | 不符合说明什么 |
|---|---|---|
| `32-bit` | 32 位可执行文件 | 出现 `64-bit` → 编译器用错 |
| `ARM` | ARM 架构 | 出现 `x86-64` → 敲成了 `gcc` |
| `interpreter /lib/ld-linux-armhf.so.3` | 动态链接 | 板子上必须有这个 loader |

**反向验证**（在 Ubuntu 上运行 ARM 程序必须失败）：

```bash
./led
# bash: ./led: cannot execute binary file: 可执行文件格式错误
```

这个报错是**正常的**，反而证明产物确实是 ARM 的。能跑起来才说明编译成了 x86。

**裁剪体积**（可选，`-g` 编译的 15 KB 降到 3.6 KB）：

```bash
arm-linux-gnueabihf-strip led
```

### 15.3.4 用 Makefile 编译（本工程已配好）

工程目录下新增了 `Makefile`：

```makefile
# 交叉编译器前缀；换工具链只改这一行
CROSS_COMPILE ?= arm-linux-gnueabihf-

CC      := $(CROSS_COMPILE)gcc
STRIP   := $(CROSS_COMPILE)strip

TARGET  := led
SRCS    := led.c
CFLAGS  := -Wall -O2 -g
LDFLAGS :=
LIBS    :=

all: $(TARGET)

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $^ -o $@ $(LDFLAGS) $(LIBS)

strip: $(TARGET)
	$(STRIP) $(TARGET)

clean:
	rm -f $(TARGET)

.PHONY: all strip clean
```

三个目标：

```bash
make            # 编译 → led
make strip      # 裁掉符号表
make clean      # 删除 led
```

> **注意**：Makefile 里命令行的缩进**必须是 TAB**，用空格会报 `missing separator`。这是初学者第一大坑。
>
> 以后编译别的例程（16_gpio、19_lcd…）把这份 Makefile 拷过去，改 `TARGET` 和 `SRCS` 就行；要切到 Yocto SDK 只改 `CROSS_COMPILE = arm-poky-linux-gnueabi-`。

### 15.3.5 编译排错表

| 现象 | 原因 | 解决 |
|---|---|---|
| `arm-linux-gnueabihf-gcc: 未找到命令` | PATH 没配 / 没生效 | `source ~/.bashrc` 或重开终端；`echo $PATH` 确认 |
| `file` 显示 `x86-64` | 敲成了 `gcc` | 用完整名 `arm-linux-gnueabihf-gcc`，`which` 确认路径 |
| 板子上 `No such file or directory`，但文件明明在 | 缺动态链接器 | 板子上 `ls /lib/ld-linux-armhf.so.3` |
| 板子上 `GLIBC_2.2x not found` | 工具链 glibc 比板子 rootfs 新 | 换匹配版本工具链，或加 `-static` 静态编译 |
| 板子上 `Permission denied` | 没有执行权限 | `chmod +x led` |
| `missing separator` | Makefile 用了空格缩进 | 改成 TAB |
| `undefined reference to 'pthread_create'` | 没链接线程库 | `-lpthread`（Makefile 里加到 `LIBS`） |
| `undefined reference to 'sin'` | 没链接数学库 | `-lm` |

---

## 15.4 在开发板上测试

启动开发板，把编译产物拷到开发板根文件系统（家目录即可），推荐用 `scp`。插图里的 `testApp` 对应本笔记的 `led`：

```bash
# 在 Ubuntu 上执行（板子 IP 换成实际的）
scp led root@192.168.1.136:/home/root/

# 板子 IP 在开发板串口里用 ifconfig eth0 查
```

> **路径坑**：ATK-IMX6U 的 `root` 家目录是 **`/home/root`**（`/etc/passwd` 里写着 `root:x:0:0:root:/home/root:/bin/sh`），登录后 `~` 就是它。传到 `/root` 也能用，但 `ls` 在家目录里看不到，容易误判"没传上去"。统一传 `/home/root/` 最省心。

![图15.4.1 testApp拷贝到开发板](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-10%20%E5%9B%BE15.4.1%20testApp%E6%8B%B7%E8%B4%9D%E5%88%B0%E5%BC%80%E5%8F%91%E6%9D%BF.png)

执行测试：

```bash
chmod +x led                # 首次拷过去要先加执行权限
./led on                    # 点亮 LED
./led off                   # 熄灭 LED
./led trigger heartbeat     # 触发模式设为 heartbeat
./led                       # 不带参数 → 打印 usage
```

![图15.4.2 测试结果](photo/%E7%AC%AC%E5%8D%81%E4%BA%94%E7%AB%A0%20%E7%82%B9%E4%BA%AELED/15-11%20%E5%9B%BE15.4.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

执行后观察板上 LED 状态是否与预期一致（on 常亮、off 熄灭、heartbeat 呼吸闪烁）。

---

## 本章速记

| 要点 | 结论 |
|---|---|
| 操控硬件两种方式 | `/dev` 设备节点（复杂设备，ioctl 为主）；`/sys` 属性文件（LED、GPIO 等简单设备） |
| sysfs 本质 | 基于内存的虚拟文件系统，内核对象→目录、对象属性→文件、对象关系→链接 |
| /sys 目录主线 | 所有设备都在 `/sys/devices`；`bus/class/dev` 是三种视角的链接组织 |
| 用户 LED 路径 | `/sys/class/leds/sys-led/`，出厂系统无 `/dev` 节点 |
| 三个属性文件 | `brightness`（读写，0 灭/非 0 亮）、`max_brightness`（只读）、`trigger`（读写，方括号为当前模式） |
| 手动控制前 | 先 `echo none > trigger` 解除触发，否则 heartbeat 等会盖掉写入 |
| 读写都是字符串 | write 属性文件必须写字符串，read 出来也是 ASCII |
| 工具链路径 | `/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf` |
| 安装包选择 | 64 位 Ubuntu 用 `x86_64` 包，32 位才用 `i686` 包（`uname -m` 判断） |
| PATH 配置 | `export PATH=$PATH:/usr/local/arm/.../bin`（**必须带 `$PATH:`**） |
| 版本验证 | `arm-linux-gnueabihf-gcc -v` → `gcc version 4.9.4 (Linaro GCC 4.9-2017.01)` |
| 编译命令 | `arm-linux-gnueabihf-gcc led.c -o led`；或 `make` |
| 产物验证 | `file led` 必须是 `32-bit ... ARM ... /lib/ld-linux-armhf.so.3` |
| 主机上跑不起来 | 正常，说明是 ARM 程序（`cannot execute binary file`） |
| 原书 Yocto SDK | `source .../environment-setup-cortexa7hf-neon-poky-linux-gnueabi` 后用 `${CC}`，**只用于 uboot/内核**，勿与应用编译混用 |
