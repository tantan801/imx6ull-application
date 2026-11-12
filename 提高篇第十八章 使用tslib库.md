# 第十八章 使用 tslib 库

> 上一章自己解析 `struct input_event` 拿坐标，麻烦且要考虑去噪、坐标变换、多点 slot 等一堆细节。**tslib 就是把这些活封装好的应用层库**：打开设备、按 `ts.conf` 加载插件（去噪、去抖、坐标变换），然后直接 `ts_read()` 拿到"已经处理过的坐标"。它是 Qt 触摸输入最常用的插件。
>
> 本章做三件事：**把 tslib 交叉编译出来 → 认识它的库函数 → 用它写单点/多点触摸程序**。板子出厂已经装了 tslib 1.16，所以移植部分主要是"自己编一遍、理解流程"，真正要写代码时直接用板上的库即可。

## 18.1 tslib 简介

tslib 是**专门为触摸屏开发的 Linux 应用层函数库**，开源，源码在 `https://github.com/libts/tslib/releases`。

它处在**触摸屏驱动与应用层之间**，充当适配层：

- 把"读取 `struct input_event` 并解析"这一整套操作封装起来，向上提供 `ts_open/ts_read` 等 API；
- 从驱动拿到原始坐标后，做**去噪、去抖、坐标变换**，把原始触摸坐标换算成屏幕坐标；
- 有一个配置文件 **`ts.conf`**，用来选择启用哪些插件、配置参数；
- 可作为 **Qt 的触摸屏输入插件**（嵌入式 Qt 项目基本都选它）。

![图18.1.1 tslib](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-1%20%E5%9B%BE18.1.1%20tslib.png)

## 18.2 tslib 移植

### 下载源码

到 tslib 的 GitHub releases 页下载：

![图18.2.1 tslib git 链接](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-2%20%E5%9B%BE18.2.1%20tslib%20git%20%E9%93%BE%E6%8E%A5.png)

**ALPHA/Mini 出厂系统已移植 tslib，版本 1.16**，可在板子上执行 `ts_finddev` 查看版本：

![图18.2.2 查看 tslib 版本](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-3%20%E5%9B%BE18.2.2%20%E6%9F%A5%E7%9C%8B%20tslib%20%E7%89%88%E6%9C%AC.png)

为了与板上一致，**也下载 1.16**：

![图18.2.3 1.16版本](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-4%20%E5%9B%BE18.2.3%201.16%E7%89%88%E6%9C%AC.png)

![图18.2.4 tslib源码包](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-5%20%E5%9B%BE18.2.4%20tslib%E6%BA%90%E7%A0%81%E5%8C%85.png)

推荐 tar.bz2 / tar.gz / tar.xz 包，原书下载的是 `tslib-1.16.tar.gz`：

![图18.2.5 tslib-1.16.tar.gz](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-6%20%E5%9B%BE18.2.5%20tslib-1.16.tar.gz.png)

> 也可以直接用光盘里的 `01、例程源码/07、第三方库源码/tslib-1.21.tar.bz2`（版本高一些，API 兼容）。

### 编译源码（三步：configure → make → make install）

把压缩包放到 Ubuntu 家目录，解压：

```bash
tar -xzf tslib-1.16.tar.gz
```

![图18.2.7 解压tslib压缩包](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-7%20%E5%9B%BE18.2.7%20%E8%A7%A3%E5%8E%8Btslib%E5%8E%8B%E7%BC%A9%E5%8C%85.png)

在家目录建 `tools/tslib` 作为**安装目录**（`--prefix` 指到这里）：

```bash
mkdir tools && cd tools && mkdir tslib
```

![图18.2.8 创建tools和tslib目录](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-8%20%E5%9B%BE18.2.8%20%E5%88%9B%E5%BB%BAtools%E5%92%8Ctslib%E7%9B%AE%E5%BD%95.png)

进入 tslib-1.16 目录：

![图18.2.9 tslib 源码](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-9%20%E5%9B%BE18.2.9%20tslib%20%E6%BA%90%E7%A0%81.png)

**第 1 步：配置工程**。先 source 交叉编译工具的环境脚本（原书用 NXP 官方 SDK）：

```bash
source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi

./configure --host=arm-poky-linux-gnueabi --prefix=/home/dt/tools/tslib/
```

两个关键选项：

| 选项         | 含义                                                                                  |
| ---------- | ----------------------------------------------------------------------------------- |
| `--host`   | 目标平台前缀，通常就是交叉编译器名称去掉 `-gcc`。`arm-poky-linux-gnueabi-gcc` → `arm-poky-linux-gnueabi` |
| `--prefix` | 库文件安装路径（这里指向刚建的 `tools/tslib`）                                                      |

`./configure --help` 可看全部选项。

![图18.2.10 配置工程](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-10%20%E5%9B%BE18.2.10%20%E9%85%8D%E7%BD%AE%E5%B7%A5%E7%A8%8B.png)

**第 2 步：编译**：

```bash
make
```

![图18.2.11 make编译](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-11%20%E5%9B%BE18.2.11%20make%E7%BC%96%E8%AF%91.png)

**第 3 步：安装**（把库、头文件、工具、配置装到 `--prefix` 指定目录）：

```bash
make install
```

![图18.2.12 安装](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-12%20%E5%9B%BE18.2.12%20%E5%AE%89%E8%A3%85.png)

### 安装目录下有什么

![图18.2.13 tslib 安装目录下的文件夹](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-13%20%E5%9B%BE18.2.13%20tslib%20%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%E5%A4%B9.png)

| 目录/文件             | 内容                                                                                         |
| ----------------- | ------------------------------------------------------------------------------------------ |
| `bin/`            | tslib 自带测试工具：`ts_print`、`ts_print_mt`、`ts_test`、`ts_test_mt`、`ts_finddev`、`ts_calibrate` 等 |
| `etc/ts.conf`     | **配置文件**，决定加载哪些插件                                                                          |
| `include/tslib.h` | **唯一的头文件**，写程序必须包含它                                                                        |
| `lib/`            | `libts.so*` 动态库；`lib/ts/` 下是插件库（`input.so`、`dejitter.so`、`linear.so` 等）                    |
| `share/`          | 可忽略                                                                                        |

![图18.2.14 bin 目录下的文件](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-14%20%E5%9B%BE18.2.14%20bin%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6.png)

`ts.conf` 的四个常用 module（取消注释即启用）：

![图18.2.15 配置文件 ts.conf 打开 ts.conf](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-15%20%E5%9B%BE18.2.15%20%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%20ts.conf%20%E6%89%93%E5%BC%80%20ts.conf.png)

![图18.2.16 ts.conf 文件的内容](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-16%20%E5%9B%BE18.2.16%20ts.conf%20%E6%96%87%E4%BB%B6%E7%9A%84%E5%86%85%E5%AE%B9.png)

| 配置行                         | 作用                                   |
| --------------------------- | ------------------------------------ |
| `module_raw input`          | **必须取消注释**，使能 input 输入事件（不启用就什么都读不到） |
| `module pthres pmin=1`      | 按压力阈值，`pmin` 调灵敏度，默认 1               |
| `module dejitter delta=100` | 去噪插件，过滤噪声样本，默认 delta=100             |
| `module linear`             | 坐标变换插件（X/Y 互换、旋转等）                   |

原书直接使用默认配置，未改动。

![图18.2.17 lib 目录](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-17%20%E5%9B%BE18.2.17%20lib%20%E7%9B%AE%E5%BD%95.png)

### 移植到开发板（自建根文件系统才需要）

出厂系统已有 tslib，**跳过这步**。若自己做根文件系统，三个拷贝动作：

```bash
# 安装目录 bin/*      → 开发板 /usr/bin/
# 安装目录 etc/ts.conf → 开发板 /etc/
# 安装目录 lib/*      → 开发板 /usr/lib/   （含 lib/ts/ 插件目录）
```

### 环境变量（tslib 靠它找设备/配置/插件）

```bash
export TSLIB_CONSOLEDEVICE=none          # 控制台设备，直接 none
export TSLIB_FBDEVICE=/dev/fb0           # 显示设备（ts_test 画线要用）
export TSLIB_TSDEVICE=/dev/input/event1  # 触摸屏设备节点，按实际填
export TSLIB_CONFFILE=/etc/ts.conf       # ts.conf 路径
export TSLIB_PLUGINDIR=/usr/lib/ts       # 插件目录
```

![图18.2.18 配置环境变量](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-18%20%E5%9B%BE18.2.18%20%E9%85%8D%E7%BD%AE%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F.png)

想开机生效就写进 `/etc/profile`。**出厂系统已配置好，通常无需设置**（详见 18.5 的实测结论）。

### 用自带工具测试

- 单点：`ts_print`（终端打印）、`ts_test`（LCD 上画线）
- 多点：`ts_print_mt`、`ts_test_mt`

```bash
ts_print        # 在触摸屏上按下/滑动/松开，终端打印坐标
```

![图18.2.19 ts\_print测试工具](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-19%20%E5%9B%BE18.2.19%20ts_print%E6%B5%8B%E8%AF%95%E5%B7%A5%E5%85%B7.png)

![图18.2.20 ts\_print\_mt 测试工具](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-20%20%E5%9B%BE18.2.20%20ts_print_mt%20%E6%B5%8B%E8%AF%95%E5%B7%A5%E5%85%B7.png)

这些工具的源码就在 tslib 源码目录的 `tests/` 里（如 `ts_test.c`）——**它们最终也是落到第 17 章那套 `input_event` 解析上**，读源码能反过来印证上章内容。

![图18.2.21 tests 文件夹](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-21%20%E5%9B%BE18.2.21%20tests%20%E6%96%87%E4%BB%B6%E5%A4%B9%20%E8%AD%AC%E5%A6%82%20ts_test%20%E7%A8%8B%E5%BA%8F%E5%AF%B9.png)

## 18.3 tslib 库函数介绍

用 tslib 编程三步：**打开设备 → 配置设备 → 读数据**。头文件只有一个：

```c
#include <tslib.h>
```

### 打开触摸屏设备

```c
#include "tslib.h"

struct tsdev *ts_open(const char *dev_name, int nonblock);
```

- `dev_name`：触摸屏设备节点；
- `nonblock`：0 = 阻塞（无数据时 read 休眠），非 0 = 非阻塞；
- 成功返回 `struct tsdev *` 句柄，失败返回 **NULL**。

另一套更常用的接口：

```c
struct tsdev *ts_setup(const char *dev_name, int nonblock);
```

区别在于：

- `dev_name` 可以传 **NULL**，此时内部读取 **`TSLIB_TSDEVICE` 环境变量**来确定设备节点；
- `ts_setup()` 除了打开设备，**还顺便做了配置**（等价于 open + `ts_config`），所以实际项目里几乎都用它。

关闭设备：

```c
int ts_close(struct tsdev *);
```

### 配置触摸屏设备

```c
#include "tslib.h"

int ts_config(struct tsdev *ts);
```

- `ts`：触摸屏句柄；
- 成功 0，失败 -1。

"配置"实质就是**解析 `ts.conf`、按需加载插件**（input / pthres / dejitter / linear）。用 `ts_setup()` 时这步已经替你做了；只有用 `ts_open()` 才需要自己再调一次。

### 读取触摸屏数据

```c
#include "tslib.h"

int ts_read(struct tsdev *ts, struct ts_sample *samp, int nr);
int ts_read_mt(struct tsdev *ts, struct ts_sample_mt **samp, int max_slots, int nr);
```

| 参数          | 说明                                                                 |
| ----------- | ------------------------------------------------------------------ |
| `ts`        | 设备句柄                                                               |
| `nr`        | 一个触摸点的采样数，**设为 1 即可**                                              |
| `max_slots` | 仅 `ts_read_mt` 用，触摸屏支持的最大触摸点数（用 ioctl `EVIOCGABS(ABS_MT_SLOT)` 获取） |

单点数据结构：

```c
struct ts_sample {
    int x;                        // X 坐标
    int y;                        // Y 坐标
    unsigned int pressure;        // 按压力大小（0 = 松开）
    struct timeval tv;            // 时间
};
```

多点数据结构（一个触摸点一个对象，组成数组传进去）：

```c
struct ts_sample_mt {
    int x;                        // X 坐标
    int y;                        // Y 坐标
    unsigned int pressure;        // 按压力大小
    int slot;                     // 触摸点 slot
    int tracking_id;              // ID
    int tool_type;
    int tool_x, tool_y;
    unsigned int touch_major, width_major, touch_minor, width_minor;
    int orientation, distance, blob_id;
    struct timeval tv;            // 时间
    short pen_down;               // BTN_TOUCH 状态
    short valid;                  // 本次采样是否有效（是否有新数据）
};
```

> 关键：**`pressure` 是判断"按下/移动/松开"的依据**（0 = 松开），**`valid` 表示该触点数据是否更新**——和第 17 章用 `BTN_TOUCH`、`TRACKING_ID` 判断是同一件事，只是被 tslib 包装好了。

## 18.4 基于 tslib 编写触摸屏应用程序

例程：光盘 `11、Linux C 应用编程例程源码 -> 18_tslib`。

### 单点触摸：ts_read.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <tslib.h>          // 包含 tslib.h 头文件

int main(int argc, char *argv[])
{
    struct tsdev *ts = NULL;
    struct ts_sample samp;
    int pressure = 0;       // 保存上一次的按压佛如力，初始 0 表示松开

    /* 打开并配置触摸屏设备 */
    ts = ts_setup(NULL, 0);
    if (NULL == ts) {
        fprintf(stderr, "ts_setup error");
        exit(EXIT_FAILURE);
    }

    /* 循环读数据 */
    for ( ; ; ) {

        if (0 > ts_read(ts, &samp, 1)) {
            fprintf(stderr, "ts_read error");
            ts_close(ts);
            exit(EXIT_FAILURE);
        }

        if (samp.pressure) {            // 按压力 > 0
            if (pressure)               // 上一次按压力也 > 0
                printf("移动(%d, %d)\n", samp.x, samp.y);
            else
                printf("按下(%d, %d)\n", samp.x, samp.y);
        }
        else
            printf("松开\n");

        pressure = samp.pressure;       // 记录本次按压力
    }

    ts_close(ts);
    exit(EXIT_SUCCESS);
}
```

状态判断逻辑（三步走）：

1. `pressure == 0` → **松开**；
2. `pressure > 0` 且**上一次** `pressure == 0` → **按下**；
3. `pressure > 0` 且上一次也 > 0 → **移动**。

`ts_read()` 出错返回负数。

**编译命令**（重点是三个参数）：

```bash
${CC} -I /home/dt/tools/tslib/include -L /home/dt/tools/tslib/lib -lts -o testApp testApp.c
```


| 参数               | 作用                                            |
| ---------------- | --------------------------------------------- |
| `-I .../include` | 头文件路径，否则找不到 `tslib.h`                         |
| `-L .../lib`     | 库文件路径                                         |
| `-lts`           | 链接 `libts.so`（Linux 动态库命名：`lib` + 名字 + `.so`） |

![图18.4.1 编译单点触摸应用程序](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-22%20%E5%9B%BE18.4.1%20%E7%BC%96%E8%AF%91%E5%8D%95%E7%82%B9%E8%A7%A6%E6%91%B8%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

拷到板子家目录运行，点击/滑动/松开触摸屏：

![图18.4.2 tslib 单点触摸应用程序测试](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-23%20%E5%9B%BE18.4.2%20tslib%20%E5%8D%95%E7%82%B9%E8%A7%A6%E6%91%B8%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E6%B5%8B%E8%AF%95.png)

### 多点触摸：ts_read_mt.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ioctl.h>
#include <linux/input.h>
#include <tslib.h>

int main(int argc, char *argv[])
{
    struct tsdev *ts = NULL;
    struct ts_sample_mt *mt_ptr = NULL;
    struct input_absinfo slot;
    int max_slots;
    unsigned int pressure[12] = {0};   // 每个触点上一次的按压力，初始 0=松开
    int i;

    /* 打开并配置触摸屏设备 */
    ts = ts_setup(NULL, 0);
    if (NULL == ts) {
        fprintf(stderr, "ts_setup error");
        exit(EXIT_FAILURE);
    }

    /* 获取最大触摸点数 */
    if (0 > ioctl(ts_fd(ts), EVIOCGABS(ABS_MT_SLOT), &slot)) {
        perror("ioctl error");
        ts_close(ts);
        exit(EXIT_FAILURE);
    }
    max_slots = slot.maximum + 1 - slot.minimum;
    printf("max_slots: %d\n", max_slots);

    /* 按最大触点数分配数组 */
    mt_ptr = calloc(max_slots, sizeof(struct ts_sample_mt));

    /* 循环读数据 */
    for ( ; ; ) {

        if (0 > ts_read_mt(ts, &mt_ptr, max_slots, 1)) {
            perror("ts_read_mt error");
            ts_close(ts);
            free(mt_ptr);
            exit(EXIT_FAILURE);
        }

        for (i = 0; i < max_slots; i++) {

            if (mt_ptr[i].valid) {                    // 有效 = 该触点有更新
                if (mt_ptr[i].pressure) {             // 按压力 > 0
                    if (pressure[mt_ptr[i].slot])     // 上一次也 > 0
                        printf("slot<%d>, 移动(%d, %d)\n",
                               mt_ptr[i].slot, mt_ptr[i].x, mt_ptr[i].y);
                    else
                        printf("slot<%d>, 按下(%d, %d)\n",
                               mt_ptr[i].slot, mt_ptr[i].x, mt_ptr[i].y);
                }
                else
                    printf("slot<%d>, 松开\n", mt_ptr[i].slot);

                pressure[mt_ptr[i].slot] = mt_ptr[i].pressure;
            }
        }
    }

    ts_close(ts);
    free(mt_ptr);
    exit(EXIT_SUCCESS);
}
```

与单点版的差异：

- 用 `ts_fd(ts)` 从 tslib 句柄取出文件描述符，再用 `ioctl(EVIOCGABS(ABS_MT_SLOT))` 求最大触点数；
- 按 `max_slots` **calloc** 一个 `ts_sample_mt` 数组，传数组地址给 `ts_read_mt`；
- 遍历数组，`valid` 为真才处理，用 `pressure[slot]` 记录上一次压力以区分按下/移动。

编译命令同上：

![图18.4.3 编译多点触摸应用程序](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-24%20%E5%9B%BE18.4.3%20%E7%BC%96%E8%AF%91%E5%A4%9A%E7%82%B9%E8%A7%A6%E6%91%B8%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

多指测试：`slot<0>` 是触摸点 0，`slot<1>` 是触摸点 1……

![图18.4.4 多点触摸应用程序测试结果](photo/%E7%AC%AC%E5%8D%81%E5%85%AB%E7%AB%A0%20%E4%BD%BF%E7%94%A8tslib%E5%BA%93/18-25%20%E5%9B%BE18.4.4%20%E5%A4%9A%E7%82%B9%E8%A7%A6%E6%91%B8%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

## 18.5 在这套环境上实操（已实测通过）

原书用的是 NXP poky SDK（`arm-poky-linux-gnueabi`）。 Ubuntu 上装的是 **Linaro 4.9.4**（`arm-linux-gnueabihf`），**一样能编**，只需把 `--host` 换成编译器前缀。

### 步骤 1：确认板子上的 tslib 状态

板子 `192.168.77.148` 上执行：

```bash
ts_finddev                      # 显示 tslib 1.16 / libts ABI version 0 (0x000901)
ls -l /usr/lib/libts.so*        # libts.so.0 -> libts.so.0.9.1
ls /usr/lib/ts                  # 插件：input.so dejitter.so linear.so ...
cat /etc/ts.conf                # module_raw input / pthres / dejitter / linear
ls /dev/input/                  # event0 event1 event2，触摸屏是 event1（goodix-ts）
```

实测结论：**出厂系统的 tslib 完整可用**，`ts_print` 直接能跑（不需要手动 export TSLIB 变量，tslib 会自动扫描设备节点；`/etc/profile.d/tslib.sh` 里那条 `TSLIB_TSDEVICE=/dev/input/touchscreen0` 因为该节点不存在，实际不生效）。

### 步骤 2：交叉编译 tslib 1.16

```bash
export PATH=$PATH:/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin

# 源码与安装目录
mkdir -p ~/linux/IMX6ULL/third_party && cd ~/linux/IMX6ULL/third_party
tar -xzf tslib-1.16.tar.gz && cd tslib-1.16

./configure --host=arm-linux-gnueabihf --prefix=$HOME/tools/tslib
make -j4
make install
```

实测产物：

```
~/tools/tslib/lib/libts.so.0.9.1:
  ELF 32-bit LSB shared object, ARM, EABI5 ... SYSV
```

**ABI 版本 `0.9.1` 与板子上的完全一致**，所以程序运行时直接复用板子的 `libts.so` 就行，不用往板子拷库。

> 说明：`configure` 会用到 libtool 的宏，但 release 包里自带 `configure` 和 `ltmain.sh`，**不需要额外装 libtool/autoconf**（我这次就没装，一步通过）。

### 步骤 3：编译例程

```bash
cd ~/linux/IMX6ULL/app-test/18_tslib
T=$HOME/tools/tslib

arm-linux-gnueabihf-gcc -I $T/include -L $T/lib -lts -o ts_read    ts_read.c
arm-linux-gnueabihf-gcc -I $T/include -L $T/lib -lts -o ts_read_mt ts_read_mt.c
```

实测产物：`ts_read` 11308 B、`ts_read_mt` 12093 B，`readelf -d` 显示 `NEEDED libts.so.0` + `libc.so.6`。

### 步骤 4：传板子并运行

> **先记住这条（踩过一次）**：ATK-IMX6U 的 `root` 用户家目录是 **`/home/root`**，不是 `/root`。
>
> ```
> /etc/passwd: root:x:0:0:root:/home/root:/bin/sh
> ```
>
> 所以登录后提示符 `~` 指的是 `/home/root`。把程序传到 `/root` 的话，`ls`（在 `~` 下）是看不到的，会误以为"没传上去"。

```bash
# Ubuntu 上：传到板子的家目录
scp ts_read ts_read_mt root@192.168.77.148:/home/root/

# 板子上（登录后就在 ~ 下，即 /home/root）
chmod +x ts_read ts_read_mt
./ts_read        # 手指按下 → "按下(x, y)"；滑动 → "移动(x, y)"；抬起 → "松开"
./ts_read_mt     # 先打印 max_slots: 5，然后每个手指一行 slot<N>
```

这边用 `timeout 3 ./ts_read` 验证过：进程会**一直阻塞等触摸**（退出码 124 = 被 timeout 杀掉，说明设备打开、读取都正常）。真实坐标需要手指去点才对得上。

### 常见报错对照

| 报错                                                     | 原因                   | 解决                                                                        |
| ------------------------------------------------------ | -------------------- | ------------------------------------------------------------------------- |
| `fatal error: tslib.h: No such file or directory`      | 没加 `-I`              | 加 `-I $HOME/tools/tslib/include`                                          |
| `undefined reference to ts_setup`                      | 没加 `-lts` 或 `-L` 路径错 | `-L $HOME/tools/tslib/lib -lts`                                           |
| 板子上 `error while loading shared libraries: libts.so.0` | 板子没有该库或路径不对          | 板子确有 `libts.so.0.9.1` 时检查 `ldd ./ts_read`；自建根文件系统需拷 `lib/`                |
| `ts_setup error`（返回 NULL）                              | 找不到设备                | 指定 `export TSLIB_TSDEVICE=/dev/input/event1`，或确认 `module_raw input` 已取消注释 |
| `ts_read error`（返回负数）                                  | 设备读失败                | 确认节点是触摸屏（`/proc/bus/input/devices` 里找 goodix-ts）、没被别的进程独占                 |
| 坐标不对/方向反了                                              | 未校准或未做坐标变换           | 用 `ts_calibrate` 生成 `/etc/pointercal`，或在 `ts.conf` 启用 `module linear` 做旋转 |
| 明明拷过去了，`ls` 却看不到                                       | 传到了 `/root`，而家目录是 `/home/root` | `ls -l /root` 确认文件在不在；要在家目录用就 `scp ...:/home/root/` |

## 本章速记

| 要点                  | 内容                                                                                                              |
| ------------------- | --------------------------------------------------------------------------------------------------------------- |
| tslib 是什么           | 触摸屏应用层库，封装 input_event 解析 + 去噪/去抖/坐标变换，可作 Qt 输入插件                                                               |
| 编程三步                | 打开设备（`ts_setup`）→ 配置（`ts_setup` 已含，或 `ts_config`）→ 读数据（`ts_read`/`ts_read_mt`）                                  |
| ts_open vs ts_setup | `ts_open` 只打开；`ts_setup` 还读 `TSLIB_TSDEVICE`（dev_name 可传 NULL）并自动 `ts_config`                                   |
| 关设备                 | `int ts_close(struct tsdev *)`                                                                                  |
| 单点结构                | `struct ts_sample {x, y, pressure, tv}`                                                                         |
| 多点结构                | `struct ts_sample_mt {x, y, pressure, slot, tracking_id, ..., valid}`                                           |
| 状态判断                | `pressure==0` 松开；`>0` 且上次 `0` 按下；`>0` 且上次 `>0` 移动                                                               |
| 多点要点                | `valid` 才处理；`slot` 区分触点；`ts_fd(ts)` 取 fd 后 `ioctl EVIOCGABS(ABS_MT_SLOT)` 求 max_slots                           |
| 移植三步                | `./configure --host=<编译器前缀> --prefix=<安装目录>` → `make` → `make install`                                          |
| 编译链接                | `-I <prefix>/include -L <prefix>/lib -lts`                                                                      |
| ts.conf 必开          | `module_raw input`；可选 `pthres`/`dejitter`/`linear`                                                              |
| 环境变量                | `TSLIB_TSDEVICE`（设备）、`TSLIB_CONFFILE`（配置）、`TSLIB_PLUGINDIR`（插件）、`TSLIB_FBDEVICE`（fb）、`TSLIB_CONSOLEDEVICE=none` |
| 自带工具                | `ts_print` / `ts_print_mt` 打印坐标；`ts_test` / `ts_test_mt` 画线；`ts_finddev` 探测；`ts_calibrate` 校准                   |
