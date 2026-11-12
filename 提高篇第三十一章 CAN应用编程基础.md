# 第三十一章 CAN 应用编程基础

> 本章分两块：**CAN 总线基础概念** + **Linux 下的 SocketCAN 编程**。
> 编程模型非常接近 TCP/UDP 那一套，只是多了"帧"和"过滤规则"的概念。

---

## 31.1 CAN 基础知识

> 本节内容参考瑞萨电子《CAN 入门教程》，资料包路径：`4、参考资料 -> CAN 入门教程.pdf`。

### 31.1.1 什么是 CAN

**CAN = Controller Area Network，控制器局域网络**，是 ISO 国际标准化的**串行通信协议**。

**起源**：德国博世（Bosch）公司开发，最初动机是解决**现代汽车中庞大电子控制系统之间的通讯、减少不断增加的信号线**。思路是设计**一条单一网络总线**，所有外围器件都挂在这条总线上。

**为什么需要它**：汽车上电子控制系统越来越多——发动机管理、变速箱、仪表、空调、车门、灯光、气囊、转向、胎压监测、制动、雷达、自适应巡航、电子防盗……这些系统之间数据类型和可靠性要求各不相同，用多条总线会导致**线束数量爆炸**。

为满足"减少线束数量""通过多个 LAN 进行大量数据高速通信"的需求，**1986 年博世开发出面向汽车的 CAN 通信协议**。此后通过 **ISO11898** 和 **ISO11519** 标准化，现在是欧洲汽车网络的标准协议，也是当前应用最广泛的**现场总线**之一。

![31-1 图31.1.1 车载网络构想.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-1%20%E5%9B%BE31.1.1%20%E8%BD%A6%E8%BD%BD%E7%BD%91%E7%BB%9C%E6%9E%84%E6%83%B3.png)

**基本设计规范**：高位置速率、高抗电磁干扰性、能检测出任何错误。

**CAN 是一种多主方式的串行通讯总线**，现已广泛应用于工业自动化、船舶、医疗设备、工业设备等领域。

### 31.1.2 CAN 的 8 个特点

**① 多主控制**

总线空闲时，**所有单元都可以开始发送**（多主控制）。
- 最先访问总线的单元获得发送权（**CSMA/CA** 方式）
- 多个单元同时发送时，**发送高优先级 ID 消息的单元获得发送权**

**② 消息的发送（ID 是优先级，不是地址）**

所有消息以固定格式发送。两个以上单元同时开始发送时，**根据标识符 ID 决定优先级**。

> ⚠️ **关键理解：ID 并不表示发送的目的地址，而是表示访问总线的消息优先级。**

仲裁方式：对各消息 ID 的**每个位逐个仲裁比较**。仲裁获胜（判定为优先级最高）的单元继续发送，**仲裁失利的单元立刻停止发送，转为接收工作**。

**③ 系统的柔软性**

挂到总线上的单元**没有"地址"信息**。所以在总线上增加单元时，其它单元的软硬件和应用层**都不需要改变**。

**④ 通信速度**

根据网络规模可设定适合的速度。

> **同一网络中所有单元必须设成统一的通信速度。** 只要有一个单元的速率和别人不一样，**这个单元就会输出错误信号，妨碍整个网络的通信**。不同网络之间可以有不同速率。

**⑤ 远程数据请求**

可通过发送**"遥控帧"**请求其它单元发送数据。

**⑥ 错误检测 / 错误通知 / 错误恢复**

- **检测**：所有单元都可以检测错误
- **通知**：检测出错误的单元立即**同时通知所有其他单元**
- **恢复**：正在发送的单元一旦检测出错误，**强制结束当前发送**，然后不断反复重发此消息直到成功

**⑦ 故障封闭**

CAN 能判断错误是**总线上暂时的数据错误**（如外部噪声）还是**持续的数据错误**（如单元内部故障、驱动器故障、断线）。

凭这个功能，当总线发生持续数据错误时，可以**把引起故障的单元从总线上隔离出去**。

**⑧ 连接**

可同时连接多个单元，**理论上单元总数无限制**。实际上受**总线时间延迟和电气负载**限制：
- **降低通信速度 → 可连单元数增加**
- **提高通信速度 → 可连单元数减少**

### 31.1.3 CAN 的电气属性

CAN 用**两根线**连接各单元：**CAN_H** 和 **CAN_L**。CAN 控制器通过判断这两根线上的**电位差**得到总线电平。

![31-2 图31.1.2 CAN 电气属性.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-2%20%E5%9B%BE31.1.2%20CAN%20%E7%94%B5%E6%B0%94%E5%B1%9E%E6%80%A7.png)

| 电平 | 逻辑 | CAN_H | CAN_L | 电位差 |
|------|------|-------|-------|--------|
| **显性电平** | **0** | 3.5V | 1.5V | **2V** |
| **隐性电平** | **1** | 2.5V | 2.5V | **0V** |

CAN 总线就通过显性/隐性电平的变化把数据发送出去。

> **总线空闲时一直处于隐性状态**（两根线都是 2.5V，差 0V）。这一点很重要——因为"显性"能覆盖"隐性"，所以多个节点同时发送时，发显性（0）的会"赢"，这就是**仲裁靠 ID 逐位比较**的物理基础。

### 31.1.4 CAN 网络拓扑

CAN 是**分布式控制总线**，网络由很多 CAN 节点构成。

![31-3 图31.1.3 CAN 网络拓扑.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-3%20%E5%9B%BE31.1.3%20CAN%20%E7%BD%91%E7%BB%9C%E6%8B%93%E6%89%91.png)

**每个节点由三部分构成**：

```
MCU（微控制器） + CAN 控制器 + CAN 收发器
                        ↓
              CAN_H ———————— CAN_H
              CAN_L ———————— CAN_L   （双绞线）
```

**要点**

- 物理介质可用**双绞线、光纤**等，**最常用双绞线**
- 信号用**差分电压**传送，两条线就是 CAN_H 和 CAN_L（开发板 CAN 接口也只有这两根）
- **CAN_H 接 CAN_H、CAN_L 接 CAN_L**（不能交叉）
- **总线两端各接一个 120Ω 端接电阻**，用于匹配总线阻抗、吸收信号反射及回拨、提高抗干扰能力和可靠性

**速率**

| 类型 | 最高速率 |
|------|---------|
| 经典 CAN | **1 Mbps** |
| CAN-FD（较新） | 最高 **5 Mbps** 甚至更高（本章不讨论） |

> **传输速度和总线距离有关：距离越短，速度越快。**

### 31.1.5 CAN 总线通信模型

CAN 传输协议参考了 **OSI 七层模型**（见 29.2 节），但**只定义了其中三层**：

| OSI 层 | CAN 是否有定义 |
|--------|--------------|
| 应用层 | ❌ 由用户/行业自定义 |
| 表示层 | ❌ |
| 会话层 | ❌ |
| **传输层** | ✅ |
| **数据链路层** | ✅（**MAC 子层是 CAN 协议的核心**） |
| **物理层** | ✅ |

![31-4 图31.1.4 OSI 七层模型和 CAN 协议.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-4%20%E5%9B%BE31.1.4%20OSI%20%E4%B8%83%E5%B1%82%E6%A8%A1%E5%9E%8B%E5%92%8C%20CAN%20%E5%8D%8F%E8%AE%AE.png)

**因为应用层没定义，所以出现了各种行业应用层协议**：

| 协议 | 用在哪 |
|------|--------|
| **DeviceNet** | 工业控制和制造业，为 PLC 和智能传感器设计 |
| **CANopen** | 工业控制 |
| **OBD** | 乘用车诊断 |
| **UDS**（统一诊断服务，ISO14229） | 乘用车诊断 |
| **SAE J1939** | 商用车 |

**数据链路层**

分 **MAC 子层**和 **LLC 子层**，**MAC 子层是 CAN 协议的核心**。

功能：把物理层收到的信号组织成有意义的消息，并提供传送错误控制等传输控制流程。具体就是**消息的帧化、仲裁、应答、错误的检测或报告**。

> 数据链路层的功能**通常在 CAN 控制器的硬件中执行**。

**物理层**

定义了信号实际的发送方式、位时序、位编码方式及同步步骤。

但**具体说，信号电平、通信速度、采样点、驱动器和总线的电气特性、连接器形态等均未定义**——这些必须由用户根据系统需求自行确定。

### 31.1.6 CAN 帧的种类

CAN 定义了 **5 种类型的报文帧**：

| 帧 | 用途 |
|----|------|
| **数据帧** | 发送单元向接收单元**传送数据**（最常用） |
| **遥控帧** | 接收单元向具有**相同 ID** 的发送单元**请求数据** |
| **错误帧** | 检测出错误时向其它单元**通知错误** |
| **过载帧** | 接收单元通知**尚未做好接收准备** |
| **间隔帧** | 把数据帧/遥控帧与前面的帧**分隔开** |

其中**数据帧和遥控帧都有两种格式**：

| 格式 | 标识符 ID 位数 |
|------|---------------|
| **标准格式** | **11 位** |
| **扩展格式** | **29 位** |

### 31.1.7 数据帧的 7 段结构

![31-5 图31.1.5 数据帧的构成.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-5%20%E5%9B%BE31.1.5%20%E6%95%B0%E6%8D%AE%E5%B8%A7%E7%9A%84%E6%9E%84%E6%88%90.png)

图中 **D = 显性电平 0**，**R = 隐性电平 1**，**D/R = 显性或隐性（0 或 1）**。

| 段 | 名称 | 作用 |
|----|------|------|
| 1 | **帧起始** | 表示数据帧开始 |
| 2 | **仲裁段** | 表示该帧**优先级**（放 ID） |
| 3 | **控制段** | 表示**数据的字节数**及保留位 |
| 4 | **数据段** | 数据内容，**可发 0~8 个字节** |
| 5 | **CRC 段** | 检查帧的**传输错误** |
| 6 | **ACK 段** | 表示**确认正常接收** |
| 7 | **帧结束** | 表示数据帧结束 |

> 更详细内容参考瑞萨《CAN 入门教程》。

---

## 31.2 SocketCAN 应用编程

**关键前提**：**Linux 把 CAN 设备当网络设备管理**。所以在 CAN 总线应用开发上，Linux 提供了 **SocketCAN** 编程接口，让 CAN 通信**近似于以太网通信**，接口更通用、更灵活。

**必须包含的头文件**：

```c
#include <linux/can.h>          /* 大部分数据结构和函数定义在这 */
#include <linux/can/raw.h>      /* CAN_RAW 相关 */
#include <net/if.h>             /* struct ifreq */
#include <sys/socket.h>
#include <sys/ioctl.h>
#include <string.h>
```

### 31.2.1 创建套接字

用标准网络套接字操作，但三个参数都是 CAN 专用的：

```c
int sockfd = -1;

sockfd = socket(PF_CAN, SOCK_RAW, CAN_RAW);
if (0 > sockfd) {
    perror("socket error");
    exit(EXIT_FAILURE);
}
```

| 参数 | 值 | 说明 |
|------|-----|------|
| `domain` | **`PF_CAN`** | 指定 CAN 通信协议 |
| `type` | **`SOCK_RAW`** | 原始套接字 |
| `protocol` | **`CAN_RAW`** | CAN 原始协议 |

### 31.2.2 将套接字与 CAN 设备绑定

CAN 设备名是 `can0`（类似网卡 `eth0`）。绑定分两步：**先查出接口索引，再 bind**。

```c
struct ifreq ifr = {0};
struct sockaddr_can can_addr = {0};
int ret;

strcpy(ifr.ifr_name, "can0");              /* 指定设备名 */
ioctl(sockfd, SIOCGIFINDEX, &ifr);         /* 用 ioctl 拿到接口索引 */

can_addr.can_family = AF_CAN;              /* 填充地址结构 */
can_addr.can_ifindex = ifr.ifr_ifindex;

ret = bind(sockfd, (struct sockaddr *)&can_addr, sizeof(can_addr));
if (0 > ret) {
    perror("bind error");
    close(sockfd);
    exit(EXIT_FAILURE);
}
```

两个结构体：

| 结构体 | 头文件 | 作用 |
|--------|--------|------|
| `struct ifreq` | `<net/if.h>` | 存接口名和接口索引 |
| `struct sockaddr_can` | `<linux/can.h>` | CAN 的地址结构（对应 `sockaddr_in`） |

> `SIOCGIFINDEX` 是 ioctl 的命令，作用是"**取接口索引**"。因为 CAN 用编号而不是名字来标识接口。

### 31.2.3 设置过滤规则

**默认行为**：不设过滤规则时，**应用会收到所有 ID 的报文**。

如果只关心特定 ID（或者只发不收），可以用 `setsockopt` 设过滤规则。

**只接收 ID 为 0x60A 和 0x60B 的帧**

```c
struct can_filter rfilter[2];

rfilter[0].can_id   = 0x60A;
rfilter[0].can_mask = 0x7FF;
rfilter[1].can_id   = 0x60B;
rfilter[1].can_mask = 0x7FF;

setsockopt(sockfd, SOL_CAN_RAW, CAN_RAW_FILTER, &rfilter, sizeof(rfilter));
```

**匹配原理**：接收条件为 `(can_id & can_mask) == (rfilter.can_id & rfilter.can_mask)`

`struct can_filter` 只有两个成员：`can_id` 和 `can_mask`。

**只发送、完全不接收**

此时可以让内核**省掉接收队列**，减少 CPU 消耗——第 4 个参数传 `NULL`、第 5 个传 `0`：

```c
setsockopt(sockfd, SOL_CAN_RAW, CAN_RAW_FILTER, NULL, 0);
```

### 31.2.4 数据收发：struct can_frame

**和普通 socket 最大的差别**：CAN 每次通信都要用 `struct can_frame` 把数据**封装成帧**。

```c
struct can_frame {
    canid_t can_id;   /* CAN 标识符 */
    __u8    can_dlc;  /* 数据长度（最长 8 个字节） */
    __u8    __pad;    /* padding */
    __u8    __res0;   /* reserved / padding */
    __u8    __res1;   /* reserved / padding */
    __u8    data[8];  /* 数据 */
};
```

**`can_id` 的用法**

| 帧类型 | 用哪几位 |
|--------|---------|
| 标准帧（SFF） | **低 11 位** |
| 扩展帧（EFF） | **0~28 位** |

**`can_id` 的第 29/30/31 位是帧标志位**，用来定义帧的类型：

```c
/* special address description flags for the CAN_ID */
#define CAN_EFF_FLAG 0x80000000U   /* 扩展帧的标识 */
#define CAN_RTR_FLAG 0x40000000U   /* 远程帧的标识 */
#define CAN_ERR_FLAG 0x20000000U   /* 错误帧的标识，用于错误检查 */

/* mask */
#define CAN_SFF_MASK 0x000007FFU   /* <can_id & CAN_SFF_MASK> 获取标准帧 ID */
#define CAN_EFF_MASK 0x1FFFFFFFU   /* <can_id & CAN_EFF_MASK> 获取扩展帧 ID */
#define CAN_ERR_MASK 0x1FFFFFFFU   /* omit EFF, RTR, ERR flags */
```

图示：

```
 bit31     bit30     bit29        bit28 ... bit0
   │         │         │              │
CAN_EFF   CAN_RTR   CAN_ERR        ID 本体（标准帧只用低 11 位）
```

### 31.2.5 数据发送

**用 `write()` 发送**。

例：帧 ID = 123，数据 3 字节 `0xA0 0xB0 0xC0`

```c
struct can_frame frame;
int ret;

frame.can_id  = 123;
frame.can_dlc = 3;             /* 数据长度为 3 */
frame.data[0] = 0xA0;
frame.data[1] = 0xB0;
frame.data[2] = 0xC0;

ret = write(sockfd, &frame, sizeof(frame));
if (sizeof(frame) != ret)      /* ret 不等于帧长度 → 发送失败 */
    perror("write error");
```

**扩展帧**就把标志位"或"上去：

```c
frame.can_id = CAN_EFF_FLAG | 123;
```

**发遥控帧**（帧 ID 为 123）：

```c
frame.can_id = CAN_RTR_FLAG | 123;
write(sockfd, &frame, sizeof(frame));
```

> **注意 `sizeof(frame)`**：整个 `struct can_frame` 大小是固定的（16 字节），**不是 `can_dlc`**。CAN 驱动要求按结构体大小收发。

### 31.2.6 数据接收

**用 `read()` 接收**：

```c
struct can_frame frame;
int ret = read(sockfd, &frame, sizeof(frame));
```

### 31.2.7 错误处理

收到一帧后，**先判断 `can_id` 里的 `CAN_ERR_FLAG` 位**，看是不是错误帧。如果是，再通过 `can_id` 的其他符号位判断具体原因。

错误符号位定义在 `<linux/can/error.h>`：

```c
/* error class (mask) in can_id */
#define CAN_ERR_TX_TIMEOUT  0x00000001U  /* TX timeout (by netdevice driver) */
#define CAN_ERR_LOSTARB     0x00000002U  /* lost arbitration / data[0] */
#define CAN_ERR_CRTL        0x00000004U  /* controller problems / data[1] */
#define CAN_ERR_PROT        0x00000008U  /* protocol violations / data[2..3] */
#define CAN_ERR_TRX         0x00000010U  /* transceiver status / data[4] */
#define CAN_ERR_ACK         0x00000020U  /* received no ACK on transmission */
#define CAN_ERR_BUSOFF      0x00000040U  /* bus off */
#define CAN_ERR_BUSERROR    0x00000080U  /* bus error (may flood!) */
#define CAN_ERR_RESTARTED   0x00000100U  /* controller restarted */
```

| 位 | 含义 | 排查方向 |
|----|------|---------|
| `TX_TIMEOUT` | 发送超时 | 总线没接、没有其它节点 ACK |
| `LOSTARB` | 仲裁失败 | 有更高优先级 ID 在发，正常现象 |
| `CRTL` | 控制器问题 | 看 `data[1]` 具体位 |
| `PROT` | 协议违规 | 看 `data[2..3]` |
| `TRX` | 收发器状态 | 看 `data[4]`，多为接线/终端电阻问题 |
| `ACK` | 发送时没收到 ACK | **典型症状：总线上只有自己一个节点** |
| `BUSOFF` | 总线关闭 | 错误累计过多，控制器自我隔离 |
| `BUSERROR` | 总线错误 | 可能刷屏 |

> **实战最常见**：只接一块板子、没接 CAN 分析仪时发送，会一直报 `CAN_ERR_ACK`（没人应答）或 `CAN_ERR_TX_TIMEOUT`。**这是正常的**，不是代码问题。

### 31.2.8 回环功能

**默认情况下 CAN 的本地回环功能是开启的**，可以用下面方法关闭/开启：

```c
int loopback = 0;   /* 0 = 关闭，1 = 开启（默认） */

setsockopt(sockfd, SOL_CAN_RAW, CAN_RAW_LOOPBACK, &loopback, sizeof(loopback));
```

**回环开启时**，所有发送帧都会被**回环到与 CAN 总线接口对应的套接字**上。

> **实用价值**：只有一块板子、没有 CAN 分析仪时，**保持 loopback = 1**，自己发的帧自己也能收到，可以单机验证收发程序。要接真实外部设备时再关掉。

---

## 31.3 CAN 应用编程实战

### 31.3.1 环境配置（必须先做）

Linux 把 CAN 当网络设备管理，**配置命令和以太网一样**。

**查看 CAN 设备**

```bash
ifconfig -a | grep can
# 或
ip link show
```

![31-6 图31.3.1 查看 can 设备.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-6%20%E5%9B%BE31.3.1%20%E6%9F%A5%E7%9C%8B%20can%20%E8%AE%BE%E5%A4%87.png)

开发板上只有**一个** CAN 接口，即 **`can0`**。

![31-7 图31.3.2 ALPHA 板上的 can 接口.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-7%20%E5%9B%BE31.3.2%20ALPHA%20%E6%9D%BF%E4%B8%8A%E7%9A%84%20can%20%E6%8E%A5%E5%8F%A3.jpg)

![31-8 图31.3.3 Mini 板上的 can 接口 在开发板出厂系统.jpg](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-8%20%E5%9B%BE31.3.3%20Mini%20%E6%9D%BF%E4%B8%8A%E7%9A%84%20can%20%E6%8E%A5%E5%8F%A3%20%E5%9C%A8%E5%BC%80%E5%8F%91%E6%9D%BF%E5%87%BA%E5%8E%82%E7%B3%BB%E7%BB%9F.jpg)

**核心配置命令（发数据前必须执行）**

```bash
ifconfig can0 down                                                  # 先关闭 can0
ip link set can0 up type can bitrate 1000000 triple-sampling on     # 波特率 1000000
```

| 参数 | 含义 |
|------|------|
| `bitrate 1000000` | 波特率 **1 Mbps** |
| `triple-sampling on` | 三采样模式（提高抗干扰） |

> ⚠️ **两个必须注意的点**：
> 1. **对端设备（CAN 分析仪 / 另一块板子）的波特率必须和开发板一致**，否则完全收不到数据，还可能出现错误帧。
> 2. 板子只有一个 CAN 接口，**自己不能和自己组网**。要么用**两块板子对接**（CANH 接 CANH、CANL 接 CANL，两端各 120Ω），要么用 **CAN 分析仪**。

### 31.3.2 用系统工具先验证硬件

出厂系统自带 `can-utils` 工具（`cansend`、`candump`）。**写代码之前先用它们确认硬件通路正常**。

**接线**：CAN 分析仪或另一块板子的 **CANH 接 CANH、CANL 接 CANL**。

![31-9 图31.3.4 CAN 上位机 设备连接好之后通过上位机软件启动.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-9%20%E5%9B%BE31.3.4%20CAN%20%E4%B8%8A%E4%BD%8D%E6%9C%BA%20%E8%AE%BE%E5%A4%87%E8%BF%9E%E6%8E%A5%E5%A5%BD%E4%B9%8B%E5%90%8E%E9%80%9A%E8%BF%87%E4%B8%8A%E4%BD%8D%E6%9C%BA%E8%BD%AF%E4%BB%B6%E5%90%AF%E5%8A%A8.png)

**测试发送**

```bash
cansend can0 123#01.02.03.04.05.06.07.08
```

格式说明：**`#` 前面是帧 ID，后面是要发的数据**（十六进制，用 `.` 分隔，最多 8 字节）。

![31-10 图31.3.5 使用 cansend 命令发送数据.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-10%20%E5%9B%BE31.3.5%20%E4%BD%BF%E7%94%A8%20cansend%20%E5%91%BD%E4%BB%A4%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE.png)

此时上位机软件能收到开发板发的数据：

![31-11 图31.3.6 上位机软件收到数据 接着测试开发板接收 CAN.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-11%20%E5%9B%BE31.3.6%20%E4%B8%8A%E4%BD%8D%E6%9C%BA%E8%BD%AF%E4%BB%B6%E6%94%B6%E5%88%B0%E6%95%B0%E6%8D%AE%20%E6%8E%A5%E7%9D%80%E6%B5%8B%E8%AF%95%E5%BC%80%E5%8F%91%E6%9D%BF%E6%8E%A5%E6%94%B6%20CAN.png)

**测试接收**

```bash
candump -ta can0
```

`-t` 显示时间戳、`-a` 用绝对时间。然后从上位机发数据：

![31-12 图31.3.7 CAN 上位机发送数据 此时开发板便能接收到上位.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-12%20%E5%9B%BE31.3.7%20CAN%20%E4%B8%8A%E4%BD%8D%E6%9C%BA%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%20%E6%AD%A4%E6%97%B6%E5%BC%80%E5%8F%91%E6%9D%BF%E4%BE%BF%E8%83%BD%E6%8E%A5%E6%94%B6%E5%88%B0%E4%B8%8A%E4%BD%8D.png)

开发板收到数据：

![31-13 图31.3.8 开发板接收到数据 测试完成之后按 Ctrl +.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-13%20%E5%9B%BE31.3.8%20%E5%BC%80%E5%8F%91%E6%9D%BF%E6%8E%A5%E6%94%B6%E5%88%B0%E6%95%B0%E6%8D%AE%20%E6%B5%8B%E8%AF%95%E5%AE%8C%E6%88%90%E4%B9%8B%E5%90%8E%E6%8C%89%20Ctrl%20%2B.png)

Ctrl+C 退出 `candump`。

> **这一步过了，说明硬件、接线、终端电阻、波特率都对**，接下来代码出问题就一定是代码问题。

### 31.3.3 CAN 数据发送实例

**功能**：每 1 秒通过 can0 发一帧，6 字节数据，帧 ID = `0x123`（标准帧）。

```c
/***************************************************************
 文件名 : can_write.c
 描述 : 一个简单地 CAN 数据发送示例代码
 ***************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <linux/can.h>
#include <linux/can/raw.h>
#include <net/if.h>

int main(void)
{
    struct ifreq ifr = {0};
    struct sockaddr_can can_addr = {0};
    struct can_frame frame = {0};
    int sockfd = -1;
    int ret;

    /* 打开套接字 */
    sockfd = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    if (0 > sockfd) {
        perror("socket error");
        exit(EXIT_FAILURE);
    }

    /* 指定 can0 设备 */
    strcpy(ifr.ifr_name, "can0");
    ioctl(sockfd, SIOCGIFINDEX, &ifr);
    can_addr.can_family = AF_CAN;
    can_addr.can_ifindex = ifr.ifr_ifindex;

    /* 将 can0 与套接字进行绑定 */
    ret = bind(sockfd, (struct sockaddr *)&can_addr, sizeof(can_addr));
    if (0 > ret) {
        perror("bind error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    /* 设置过滤规则：不接受任何报文、仅发送数据 */
    setsockopt(sockfd, SOL_CAN_RAW, CAN_RAW_FILTER, NULL, 0);

    /* 填充要发送的数据 */
    frame.data[0] = 0xA0;
    frame.data[1] = 0xB0;
    frame.data[2] = 0xC0;
    frame.data[3] = 0xD0;
    frame.data[4] = 0xE0;
    frame.data[5] = 0xF0;
    frame.can_dlc = 6;        /* 一次发送 6 个字节数据 */
    frame.can_id  = 0x123;    /* 帧 ID 为 0x123，标准帧 */

    for ( ; ; ) {
        ret = write(sockfd, &frame, sizeof(frame));
        if (sizeof(frame) != ret) {   /* ret 不等于帧长度 → 发送失败 */
            perror("write error");
            goto out;
        }

        sleep(1);             /* 一秒钟发送一次 */
    }

out:
    close(sockfd);
    exit(EXIT_SUCCESS);
}
```

**编译**

```bash
. /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
${CC} -o can_write can_write.c
```

![31-14 图31.3.9 编译示例代码 将编译得到的可执行文件 can_w.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-14%20%E5%9B%BE31.3.9%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81%20%E5%B0%86%E7%BC%96%E8%AF%91%E5%BE%97%E5%88%B0%E7%9A%84%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%20can_w.png)

**运行**

```bash
scp can_write root@192.168.77.148:/home/root/

# 板子上
ifconfig can0 down
ip link set can0 up type can bitrate 1000000 triple-sampling on
./can_write
```

上位机就能收到每 1 秒一帧、ID=`0x123`、6 字节的数据：

![31-15 图31.3.10 CAN 上位机接收到数据.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-15%20%E5%9B%BE31.3.10%20CAN%20%E4%B8%8A%E4%BD%8D%E6%9C%BA%E6%8E%A5%E6%94%B6%E5%88%B0%E6%95%B0%E6%8D%AE.png)

### 31.3.4 CAN 数据接收实例

**功能**：循环读帧，打印帧类型（标准/扩展）、帧格式（数据/远程）、数据长度和内容。

```c
/***************************************************************
 文件名 : can_read.c
 描述 : 一个简单地 CAN 数据读取示例代码
 ***************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <linux/can.h>
#include <linux/can/raw.h>
#include <net/if.h>

int main(void)
{
    struct ifreq ifr = {0};
    struct sockaddr_can can_addr = {0};
    struct can_frame frame = {0};
    int sockfd = -1;
    int i;
    int ret;

    /* 打开套接字 */
    sockfd = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    if (0 > sockfd) {
        perror("socket error");
        exit(EXIT_FAILURE);
    }

    /* 指定 can0 设备 */
    strcpy(ifr.ifr_name, "can0");
    ioctl(sockfd, SIOCGIFINDEX, &ifr);
    can_addr.can_family = AF_CAN;
    can_addr.can_ifindex = ifr.ifr_ifindex;

    /* 将 can0 与套接字进行绑定 */
    ret = bind(sockfd, (struct sockaddr *)&can_addr, sizeof(can_addr));
    if (0 > ret) {
        perror("bind error");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    /* 设置过滤规则 */
    /* 本例程要接收所有报文，所以不能设"只发"过滤；想只收特定 ID 就放开下面这行 */
    /* setsockopt(sockfd, SOL_CAN_RAW, CAN_RAW_FILTER, rfilter, sizeof(rfilter)); */

    /* 接收数据 */
    for ( ; ; ) {
        if (0 > read(sockfd, &frame, sizeof(struct can_frame))) {
            perror("read error");
            break;
        }

        /* 校验是否接收到错误帧 */
        if (frame.can_id & CAN_ERR_FLAG) {
            printf("Error frame!\n");
            break;
        }

        /* 校验帧格式：扩展帧还是标准帧 */
        if (frame.can_id & CAN_EFF_FLAG)
            printf("扩展帧 <0x%08x> ", frame.can_id & CAN_EFF_MASK);
        else
            printf("标准帧 <0x%03x> ", frame.can_id & CAN_SFF_MASK);

        /* 校验帧类型：远程帧还是数据帧 */
        if (frame.can_id & CAN_RTR_FLAG) {
            printf("remote request\n");
            continue;
        }

        /* 打印数据长度 */
        printf("[%d] ", frame.can_dlc);

        /* 打印数据 */
        for (i = 0; i < frame.can_dlc; i++)
            printf("%02x ", frame.data[i]);
        printf("\n");
    }

    close(sockfd);
    exit(EXIT_SUCCESS);
}
```

**编译**

```bash
${CC} -o can_read can_read.c
```

![31-16 图31.3.11 编译示例代码.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-16%20%E5%9B%BE31.3.11%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

**运行**

先按 31.3.1 配好 can0，然后：

```bash
scp can_read root@192.168.77.148:/home/root/
./can_read
```

上位机发数据，板上打印：

![31-17 图31.3.12 开发板接收到上位机的数据.png](photo/%E7%AC%AC%E4%B8%89%E5%8D%81%E4%B8%80%E7%AB%A0%20CAN%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/31-17%20%E5%9B%BE31.3.12%20%E5%BC%80%E5%8F%91%E6%9D%BF%E6%8E%A5%E6%94%B6%E5%88%B0%E4%B8%8A%E4%BD%8D%E6%9C%BA%E7%9A%84%E6%95%B0%E6%8D%AE.png)

输出形如：

```
标准帧 <0x123> [8] 01 02 03 04 05 06 07 08
```

---

## 常见坑（CAN 调试照这个查）

| 现象 | 原因 | 解决 |
|------|------|------|
| `bind error: No such device` | `can0` 不存在，或设备没起来 | 先 `ifconfig can0 up`；内核没编 CAN 驱动则节点根本没生成 |
| **能发但对方收不到** | **波特率不一致**（最常见） | 两边都设 `bitrate 1000000`，核对分析仪设置 |
| 发送时一直报 `CAN_ERR_ACK` | 总线上**只有自己一个节点**，没人 ACK | 接上第二个节点；或开 loopback 单机测试 |
| `write error: Network is down` | can0 没 up | `ip link set can0 up type can bitrate ...` |
| 收到一堆错误帧刷屏 | 接线反了（CANH/CANL 交叉）、没接终端电阻、波特率错 | 检查 CANH↔CANH、两端 120Ω |
| 只能收到部分 ID 的帧 | 设了 `CAN_RAW_FILTER` 过滤 | 删掉过滤或放宽 `can_mask` |
| 单板自测收不到自己发的 | loopback 被关了 | 保持默认（开）或显式设 `loopback=1` |
| 数据长度不对 | `can_dlc` 忘了设 / 超过 8 | `can_dlc` 最大 **8** |

---

## 本章小结

**CAN 概念速查**

| 项 | 值 |
|----|-----|
| 全称 | Controller Area Network，控制器局域网络 |
| 出身 | 1986 博世，ISO11898 / ISO11519 标准化 |
| 拓扑 | 总线型，**双绞线 CAN_H / CAN_L** |
| 终端电阻 | **总线两端各 120Ω** |
| 电平 | **显性 = 0**（H 3.5V / L 1.5V，差 2V）；**隐性 = 1**（都 2.5V，差 0V）；**空闲为隐性** |
| 速率 | 经典 1 Mbps；CAN-FD 最高 5 Mbps；**距离越短越快** |
| 寻址 | **ID 是优先级，不是地址**；标准帧 11 位 / 扩展帧 29 位 |
| 帧种类 | 数据帧、遥控帧、错误帧、过载帧、间隔帧 |
| 数据段 | **0~8 字节** |

**SocketCAN 编程 4 步**

```
socket(PF_CAN, SOCK_RAW, CAN_RAW)
    ↓
ioctl(SIOCGIFINDEX) 取 can0 索引 → bind()
    ↓
setsockopt(CAN_RAW_FILTER) 设过滤（只发就传 NULL,0）
    ↓
write/read 收发 struct can_frame
```

**和 TCP socket 的对照**

| | TCP socket | SocketCAN |
|--|-----------|-----------|
| domain | `AF_INET` | **`PF_CAN`** |
| type | `SOCK_STREAM` | **`SOCK_RAW`** |
| protocol | `0` | **`CAN_RAW`** |
| 地址结构 | `sockaddr_in` | **`sockaddr_can`**（靠 `ifindex` 而非 IP） |
| 绑定前 | 不需要额外查询 | **必须先 `ioctl(SIOCGIFINDEX)` 查索引** |
| 收发单位 | 字节流（无边界） | **`struct can_frame`（有帧边界）** |

> CAN 在汽车电子和工业领域用得极广。本章和上一章一样，原书定位是**引导入门**。
