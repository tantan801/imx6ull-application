# 第二十八章 ALSA 音频应用编程

> 本章目标：用 alsa-lib 在 I.MX6U 上完成 **播放 WAV、录制音频、异步播放、poll 播放、混音器调音量**。
> 全章只需要三个模块：**PCM Interface（核心）、Error Interface、Mixer Interface**。

---

## 28.1 ALSA 概述

**ALSA = Advanced Linux Sound Architecture**，Linux 下主流的音频体系架构，替代了老的 OSS，提供音频和 MIDI 支持。

它是内核里的一套音频驱动框架，设计上分了层，应用层不用管驱动细节——ALSA 在用户空间提供了标准的 C 库 **alsa-lib**，应用程序调这套 API 就能完成播放、录音。

![28-1 图28.1.1 alsa 音频示意.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-1%20%E5%9B%BE28.1.1%20alsa%20%E9%9F%B3%E9%A2%91%E7%A4%BA%E6%84%8F.png)

一句话概括分层：

```
应用程序 → alsa-lib（用户态库）→ 内核 ALSA 驱动 → WM8960 编解码芯片 → 喇叭/耳机/麦克风
```

---

## 28.2 alsa-lib 简介

alsa-lib 是一套 **Linux 应用层 C 语言函数库**，为音频应用提供统一、标准的接口。它的价值是**屏蔽驱动细节**，应用不需要直接读写音频设备节点。

官方在线文档：`https://www.alsa-project.org/alsa-doc/alsa-lib/`

![28-2 图28.2.1 alsa-lib.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-2%20%E5%9B%BE28.2.1%20alsa-lib.png)

### 模块划分

点文档里的 **Modules** 能看到全部模块，带三角箭头的模块下面还有子模块。

![28-3 图28.2.2 alsa-lib 模块.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-3%20%E5%9B%BE28.2.2%20alsa-lib%20%E6%A8%A1%E5%9D%97.png)

| 模块 | 作用 |
|------|------|
| Global defines and functions | 全局函数、宏 |
| Constants for Digital Audio Interfaces | 数字音频接口常量 |
| Input / Output Interface | 输入 / 输出接口 |
| Error handling | 错误处理（`snd_strerror`） |
| Configuration Interface | 配置接口 |
| **Control Interface** | 控制接口 |
| **PCM Interface** | **PCM 设备接口（本章核心）** |
| RawMidi / MIDI Sequencer | MIDI |
| Timer Interface | 定时器 |
| Hardware Dependant Interface | 硬件相关 |
| **Mixer Interface** | **混音器（音量、声道、增益）** |
| Use Case / Topology Interface | 用例 / 拓扑 |

模块很多，本章**只涉及三个**：**PCM Interface、Error Interface、Mixer Interface**。

### PCM Interface

提供 PCM 设备相关操作：打开/关闭设备、配置硬件或软件参数、控制设备（启动、暂停、恢复、读写数据）。

![28-4 图28.2.3 PCM Interface 下的子模块 点击模块.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-4%20%E5%9B%BE28.2.3%20PCM%20Interface%20%E4%B8%8B%E7%9A%84%E5%AD%90%E6%A8%A1%E5%9D%97%20%E7%82%B9%E5%87%BB%E6%A8%A1%E5%9D%97.png)

- **Error Interface**：出错时用 `snd_strerror()` 打印描述，用法同 C 库 `strerror()`。
- **Mixer Interface**：音量、声道控制、增益等。

---

## 28.3 sound 设备节点

基于 ALSA 框架注册的声卡，会在 **`/dev/snd/`** 下生成设备节点。

![28-5 图28.3.1 devsnd 目录下的文件 Tips.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-5%20%E5%9B%BE28.3.1%20devsnd%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%20Tips.png)

命名规则：`pcmC<卡号>D<设备号><c|p>`，`c` = **c**apture 录音，`p` = **p**layback 播放。

| 节点 | 含义 |
|------|------|
| `controlC0` | 声卡 0 的控制设备（通道选择、混音器、麦克风控制） |
| `pcmC0D0c` | 声卡 0 / 设备 0 的**录音**设备 |
| `pcmC0D0p` | 声卡 0 / 设备 0 的**播放**设备 |
| `pcmC0D1c` | 声卡 0 / 设备 1 的录音设备 |
| `pcmC0D1p` | 声卡 0 / 设备 1 的播放设备 |
| `timer` | 定时器 |

> **Mini 板没有音频芯片，`/dev/snd` 是空的**，本章例程跑不了。ALPHA 板才有 WM8960。

程序虽然走 alsa-lib，最终仍落实到对 sound 节点的 I/O 操作，只是库帮你封装了。

### /proc/asound

![28-6 图28.3.2 procasound 目录下的文件.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-6%20%E5%9B%BE28.3.2%20procasound%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6.png)

```bash
cat /proc/asound/cards      # 系统里注册了哪些声卡
cat /proc/asound/devices    # 所有声卡注册的设备（control/pcm/timer/seq）
cat /proc/asound/pcm        # 所有 PCM 设备（playback + capture）
```

![28-7 图28.3.3 查看系统中注册的所有声卡.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-7%20%E5%9B%BE28.3.3%20%E6%9F%A5%E7%9C%8B%E7%B3%BB%E7%BB%9F%E4%B8%AD%E6%B3%A8%E5%86%8C%E7%9A%84%E6%89%80%E6%9C%89%E5%A3%B0%E5%8D%A1.png)

ALPHA 板只有一个声卡（WM8960），编号 0。每块声卡在 `/proc/asound/` 下对应一个 `cardX` 目录。

![28-8 图28.3.4 card0 目录下的文件.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-8%20%E5%9B%BE28.3.4%20card0%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6.png)

![28-9 图28.3.5 列出所有设备.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-9%20%E5%9B%BE28.3.5%20%E5%88%97%E5%87%BA%E6%89%80%E6%9C%89%E8%AE%BE%E5%A4%87.png)

![28-10 图28.3.6 列出系统中所有 PCM 设备.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-10%20%E5%9B%BE28.3.6%20%E5%88%97%E5%87%BA%E7%B3%BB%E7%BB%9F%E4%B8%AD%E6%89%80%E6%9C%89%20PCM%20%E8%AE%BE%E5%A4%87.png)

---

## 28.4 alsa-lib 移植 + alsa-utils 工具

### 28.4.1 原书的做法

原书说得很直接：**ALPHA 出厂系统已经移植好了 alsa-lib 和 alsa-utils**，直接拿来用，不再介绍移植过程。

而且还有一个更省事的点——**交叉编译工具的 sysroot 里本来就带了 alsa-lib**：

| 内容 | sysroot 路径 |
|------|-------------|
| 头文件 | `<sysroot>/usr/include/alsa/asoundlib.h` |
| 库文件 | `<sysroot>/usr/lib/libasound.so` → `libasound.so.2.0.0` |

`${CC}` 里带了 `--sysroot=/opt/fsl-imx-x11/4.1.15-2.1.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi`，编译器会自动去 `<sysroot>/usr/include` 找头文件、去 `<sysroot>/lib` 和 `<sysroot>/usr/lib` 找库。

所以**最简编译命令只要一个 `-lasound`**：

```bash
${CC} -o pcm_playback pcm_playback.c -lasound
```

### 28.4.2 自己动手移植 alsa-lib（本次实操）

想版本独立、或板子系统里没有 alsa-lib 时需要自己移植。完整步骤如下。

**① 下载源码**（官方地址）

```bash
cd ~/linux/IMX6ULL/third_party
wget https://www.alsa-project.org/files/pub/lib/alsa-lib-1.1.5.tar.bz2
# 校验完整性，包约 957K。下载中断会得到 500~600K 的残缺包，必须校验！
bzip2 -t alsa-lib-1.1.5.tar.bz2      # 无输出即正常
tar -xjf alsa-lib-1.1.5.tar.bz2
```

**② 交叉配置**

```bash
. /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
cd ~/linux/IMX6ULL/third_party/alsa-lib-1.1.5
./configure --host=arm-poky-linux-gnueabi \
            --prefix=$HOME/tools/alsa \
            --disable-static --enable-shared --disable-python \
            --with-alsa-devdir=/dev/snd
```

参数说明：

| 参数 | 作用 |
|------|------|
| `--host=arm-poky-linux-gnueabi` | 交叉编译目标 |
| `--prefix=$HOME/tools/alsa` | 安装目录，**不要装到 /usr**，避免污染宿主机 |
| `--disable-static` | 只出 .so，省空间 |
| `--disable-python` | 不编 python 绑定 |
| `--with-alsa-devdir=/dev/snd` | 设备节点目录 |

配置成功的判据：`config.log` 里 `CC` 是 `arm-poky-linux-gnueabi-gcc ... --sysroot=...`。

**③ 编译安装**

```bash
make -j4 && make install
ls $HOME/tools/alsa/lib        # libasound.so.2.0.0 + 两个符号链接
```

**④ 精简体积**

```bash
arm-poky-linux-gnueabi-strip $HOME/tools/alsa/lib/libasound.so.2.0.0
# 3.8MB → 840KB
```

**⑤ 部署到板子（独立目录，绝不覆盖系统库）**

```bash
cd $HOME/tools/alsa
tar -czf /tmp/alsa_lib.tar.gz   lib/libasound.so*
tar -czf /tmp/alsa_share.tar.gz -C share alsa
scp /tmp/alsa_lib.tar.gz /tmp/alsa_share.tar.gz root@192.168.77.148:/home/root/

# 板子上
mkdir -p /home/root/alsa_lib
tar -xzf /home/root/alsa_lib.tar.gz   -C /home/root/alsa_lib      # → alsa_lib/lib/
tar -xzf /home/root/alsa_share.tar.gz -C /home/root/alsa_lib      # → alsa_lib/alsa/，需改成 share/alsa
mkdir -p /home/root/alsa_lib/share
mv /home/root/alsa_lib/alsa /home/root/alsa_lib/share/alsa
```

最终板子上的结构：

```
/home/root/alsa_lib/
├── lib/
│   ├── libasound.so      -> libasound.so.2.0.0
│   ├── libasound.so.2    -> libasound.so.2.0.0
│   └── libasound.so.2.0.0     (859212 字节，ARM 32-bit)
└── share/alsa/
    ├── alsa.conf
    ├── alsa.conf.d/  cards/  pcm/  topology/  ucm/
```

> ⚠️ **`share/alsa/alsa.conf` 不能漏**。alsa-lib 打开 PCM 时第一件事就是读这个配置文件，找不到会报 `cannot open configuration file`。用自己移植的库时，要么把配置放到 `/usr/share/alsa/`，要么设 `ALSA_CONFIG_PATH`。

### 28.4.3 alsa-utils 工具

alsa-utils 是基于 alsa-lib 写的测试/配置工具集，出厂系统里都有。

**aplay —— 播放 wav**

```bash
aplay test.wav
```

ALPHA 板支持喇叭/耳机自动切换：没插耳机走喇叭，插上耳机自动切耳机。**aplay 只认 wav，不认 mp3**（没做 mp3 解码）。

![28-11 图28.4.1 使用 aplay 播放 wav 音乐.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-11%20%E5%9B%BE28.4.1%20%E4%BD%BF%E7%94%A8%20aplay%20%E6%92%AD%E6%94%BE%20wav%20%E9%9F%B3%E4%B9%90.png)

**alsamixer —— 字符图形化混音器**

```bash
alsamixer
```

![28-12 图28.4.2 alsamixer 界面.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-12%20%E5%9B%BE28.4.2%20alsamixer%20%E7%95%8C%E9%9D%A2.png)

- 左上 `Card: wm8960-audio` = 当前声卡，多声卡按 **F6** 切换
- 按 **H** 看操作说明
- **F3** Playback / **F4** Capture / **F5** All
- 上下键调值，M 键开关（switch 类配置项）

> ⚠️ 在串口终端按 F4/F5 可能直接退出界面，是快捷键被别的程序抢了。用 Ubuntu 通过 **ssh 远程登录**再执行 alsamixer 就正常。

![28-13 图28.4.3 alsamixer 界面操作说明 不同声卡支持的.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-13%20%E5%9B%BE28.4.3%20alsamixer%20%E7%95%8C%E9%9D%A2%E6%93%8D%E4%BD%9C%E8%AF%B4%E6%98%8E%20%E4%B8%8D%E5%90%8C%E5%A3%B0%E5%8D%A1%E6%94%AF%E6%8C%81%E7%9A%84.png)

**WM8960 常用配置项**

| 配置项 | 含义 |
|--------|------|
| `Headphone` | 耳机音量（0~127），Q/Z 调左声道，E/C 调右声道 |
| `Headphone Playback ZC` | 耳机播放 ZC，M 键开关 |
| `Speaker` | 喇叭音量 |
| `Speaker AC` / `Speaker DC` | 喇叭 ZC / DC |
| `Speaker Playback ZC` | 喇叭播放 ZC，M 键开关 |
| `Playback` | 播放音量，**同时作用于喇叭和耳机** |
| `Capture` | 采集（录音）音量 |

**asound.state —— 声卡配置持久化**

出厂系统里有 `/var/lib/alsa/asound.state`，每次开机自动加载、关机自动保存（由 alsactl 完成）。直接关机的话，之前临时改的设置会丢。

```bash
alsactl -f /var/lib/alsa/asound.state store      # 保存当前配置
alsactl -f /var/lib/alsa/asound.state restore    # 加载配置
```

![28-14 图28.4.4 asound.state 文件部分内容 除了保存.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-14%20%E5%9B%BE28.4.4%20asound.state%20%E6%96%87%E4%BB%B6%E9%83%A8%E5%88%86%E5%86%85%E5%AE%B9%20%E9%99%A4%E4%BA%86%E4%BF%9D%E5%AD%98.png)

**amixer —— 命令行混音器**

```bash
amixer scontrols                      # 看有哪些配置项
amixer scontents                      # 看每个配置项的说明和当前值
amixer sset Headphone 100,100         # 耳机音量左右都设 100
amixer sset "Headphone Playback ZC" on
amixer sset "Left Output Mixer PCM" on
```

![28-15 图28.4.5 查看有哪些配置项.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-15%20%E5%9B%BE28.4.5%20%E6%9F%A5%E7%9C%8B%E6%9C%89%E5%93%AA%E4%BA%9B%E9%85%8D%E7%BD%AE%E9%A1%B9.png)

![28-16 图28.4.6 每一个配置项的配置说明 “Headphone”配.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-16%20%E5%9B%BE28.4.6%20%E6%AF%8F%E4%B8%80%E4%B8%AA%E9%85%8D%E7%BD%AE%E9%A1%B9%E7%9A%84%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E%20%E2%80%9CHeadphone%E2%80%9D%E9%85%8D.png)

**arecord —— 录音**

```bash
arecord -f cd -d 10 test.wav
# -f cd   = 16bit little endian, 44100, stereo
# -d 10   = 录 10 秒
```

![28-17 图28.4.7 使用 arecord 工具录音.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-17%20%E5%9B%BE28.4.7%20%E4%BD%BF%E7%94%A8%20arecord%20%E5%B7%A5%E5%85%B7%E5%BD%95%E9%9F%B3.png)

### 28.4.4 播放前的声卡配置（重要）

```bash
amixer sset 'Headphone Playback ZC' on
amixer sset 'Speaker Playback ZC' on
amixer sset 'Speaker AC' 3
amixer sset 'Speaker DC' 3
amixer sset Headphone 105,105
amixer sset Playback 230,230
amixer sset Speaker 118,118
amixer sset 'Right Output Mixer PCM' on
amixer sset 'Left Output Mixer PCM' on
```

出厂系统一般已经配好，但自己动手改过 alsamixer 的话，这里是最小可用配置。

![28-28 图28.5.11 声卡设置 由于篇幅有限打印信息不能给.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-28%20%E5%9B%BE28.5.11%20%E5%A3%B0%E5%8D%A1%E8%AE%BE%E7%BD%AE%20%E7%94%B1%E4%BA%8E%E7%AF%87%E5%B9%85%E6%9C%89%E9%99%90%E6%89%93%E5%8D%B0%E4%BF%A1%E6%81%AF%E4%B8%8D%E8%83%BD%E7%BB%99.png)

---

## 28.5 编写一个简单的 alsa-lib 应用程序

动手前先记住两份官方资料，比翻任何教程都准：

- 入门教程：`https://users.suse.com/~mana/alsa090_howto.html`（PCM 播放、录音都有，原书就是参考它写的）
- 示例代码：`https://www.alsa-project.org/alsa-doc/alsa-lib/examples.html`

点文件名就能看源码：

![28-18 图28.5.1 ALSA 提供的参考代码 点击对应源文件即可查看.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-18%20%E5%9B%BE28.5.1%20ALSA%20%E6%8F%90%E4%BE%9B%E7%9A%84%E5%8F%82%E8%80%83%E4%BB%A3%E7%A0%81%20%E7%82%B9%E5%87%BB%E5%AF%B9%E5%BA%94%E6%BA%90%E6%96%87%E4%BB%B6%E5%8D%B3%E5%8F%AF%E6%9F%A5%E7%9C%8B.png)

### 28.5.1 先搞清楚 7 个概念

| 概念 | 含义 | 举例 |
|------|------|------|
| **样本长度 Sample** | 采样位数 / 位深度（Bit Depth、Sample Size、Sample Width），每个样本多少 bit | 8bit、16bit、24bit |
| **声道数 channel** | 单声道 1、立体声 2 | 2 |
| **帧 frame** | 一个声音单元 = 样本长度 × 声道数 | 16bit 双声道 = 16×2/8 = **4 字节** |
| **采样率 Sample rate** | 每秒采样多少次（针对帧） | 8K 电话、22.05K FM、44.1K CD、48K DVD |
| **交错模式 interleaved** | 数据按"帧 1 左、帧 1 右、帧 2 左、帧 2 右…"连续存放 | 绝大多数情况都用这个 |
| **周期 period** | 设备一次读/写的数据单位，含若干帧；也是**两次硬件中断之间的帧数** | 1024 帧 = 1024×4 = 4096 字节 |
| **缓冲区 buffer** | 由若干周期组成，DMA 搬运的环形缓冲区 | 16 个周期 |

**非交错模式**：先存一个周期内所有帧的左声道，再存所有右声道。用了非交错就不能用 `readi/writei`，得用 `readn/writen`。

![28-19 图28.5.2 bufferperiodframesample.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-19%20%E5%9B%BE28.5.2%20bufferperiodframesample.png)

**为什么要分周期而不是一次搬完整个 buffer？**

因为**延迟**。buffer 越大，一次搬完耗时越长，声音从"开始传输"到"发出来"的间隔就越长。
但周期也不能太小——太小会频繁触发中断，CPU 一直被打断，效率下降。
**结论**：延迟可接受的前提下，周期尽量设大一些；低延迟实时场景再往小调。

### 28.5.2 buffer 里数据怎么流动

**播放**：应用 `write` → 应用往 buffer 写（write pointer 前进）；设备 DMA 从 buffer 读（read pointer 前进）。buffer 满时应用写不进去，要等设备播完一个周期腾出空位。

**录音**：设备 DMA 往 buffer 写（write pointer 前进）；应用从 buffer 读（read pointer 前进）。buffer 空时应用读不到，要等设备采满一个周期。

两个指针走到 buffer 末尾都会回到开头，**这是环形缓冲区**。

![28-20 图28.5.3 pointer 指向 buffer 起始位置 应.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-20%20%E5%9B%BE28.5.3%20pointer%20%E6%8C%87%E5%90%91%20buffer%20%E8%B5%B7%E5%A7%8B%E4%BD%8D%E7%BD%AE%20%E5%BA%94.png)

![28-21 图28.5.4 buffer 中数据的传输.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-21%20%E5%9B%BE28.5.4%20buffer%20%E4%B8%AD%E6%95%B0%E6%8D%AE%E7%9A%84%E4%BC%A0%E8%BE%93.png)

### 28.5.3 Overrun / Underrun（XRUN）

- **overrun**（录音）：应用读得不够快，buffer 被设备写满，数据被覆盖丢失。
- **underrun**（播放）：应用写得不够快，buffer 被设备读空，没数据可播（缓冲"饿死"）。

两者合称 **XRUN**。合理设计周期大小和应用节奏可以把 XRUN 降到最低，并能从中恢复（`snd_pcm_prepare()`）。

### 28.5.4 打开 PCM 设备

```c
#include <alsa/asoundlib.h>

int snd_pcm_open(snd_pcm_t **pcmp, const char *name,
                 snd_pcm_stream_t stream, int mode);
int snd_pcm_close(snd_pcm_t *pcm);
```

| 参数 | 说明 |
|------|------|
| `pcmp` | 返回 PCM 设备句柄（`snd_pcm_t` 描述一个 PCM 设备） |
| `name` | **逻辑设备名**，不是设备节点名。格式 `"hw:i,j"`，i=卡号，j=设备号。`"hw:0,0"` 播放对应 `/dev/snd/pcmC0D0p`，录音对应 `pcmC0D0c` |
| `stream` | `SND_PCM_STREAM_PLAYBACK` 播放 / `SND_PCM_STREAM_CAPTURE` 采集 |
| `mode` | `0` = 阻塞（默认）；`SND_PCM_NONBLOCK` = 非阻塞 |

成功返回 0，失败返回负错误码，用 `snd_strerror(ret)` 转描述。

```c
snd_pcm_t *pcm_handle = NULL;
int ret;

ret = snd_pcm_open(&pcm_handle, "hw:0,0", SND_PCM_STREAM_PLAYBACK, 0);
if (0 > ret) {
    fprintf(stderr, "snd_pcm_open error: %s\n", snd_strerror(ret));
    return -1;
}
```

### 28.5.5 设置硬件参数（7 步）

先实例化 `snd_pcm_hw_params_t`：

```c
snd_pcm_hw_params_t *hwparams = NULL;
snd_pcm_hw_params_malloc(&hwparams);       // 或 snd_pcm_hw_params_alloca()
snd_pcm_hw_params_any(pcm, hwparams);      // 用设备当前配置初始化
```

然后逐项设置，最后生效。

**① 访问类型**

```c
int snd_pcm_hw_params_set_access(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, snd_pcm_access_t access);
```

```c
enum snd_pcm_access_t {
    SND_PCM_ACCESS_MMAP_INTERLEAVED = 0,
    SND_PCM_ACCESS_MMAP_NONINTERLEAVED,
    SND_PCM_ACCESS_MMAP_COMPLEX,
    SND_PCM_ACCESS_RW_INTERLEAVED,      /* 最常用：配合 readi/writei */
    SND_PCM_ACCESS_RW_NONINTERLEAVED,   /* 配合 readn/writen */
    SND_PCM_ACCESS_LAST = SND_PCM_ACCESS_RW_NONINTERLEAVED
};
```

**② 数据格式**

```c
int snd_pcm_hw_params_set_format(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, snd_pcm_format_t format);
```

枚举很长，最常用 **`SND_PCM_FORMAT_S16_LE`**（有符号 16 位、小端）。
不确定设备支不支持，可以先测：

```c
if (snd_pcm_hw_params_test_format(pcm, hwparams, SND_PCM_FORMAT_S16_LE))
    /* 返回非 0 = 不支持 */
else
    /* 返回 0 = 支持 */
```

**③ 声道数**

```c
int snd_pcm_hw_params_set_channels(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, unsigned int val);   /* val=2 立体声 */
```

**④ 采样率**

```c
int snd_pcm_hw_params_set_rate(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, unsigned int val, int dir);
```

`dir` 控制方向：`-1` 实际值小于 val、`0` 等于 val、`1` 大于 val。

**⑤ 周期大小（单位是帧，不是字节）**

```c
int snd_pcm_hw_params_set_period_size(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, snd_pcm_uframes_t val, int dir);
```

**⑥ buffer 大小（两种写法）**

```c
/* 按帧：16 个周期 × 1024 帧 = 16384 帧 */
int snd_pcm_hw_params_set_buffer_size(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, snd_pcm_uframes_t val);

/* 按周期数（注意单位不同！）*/
int snd_pcm_hw_params_set_periods(snd_pcm_t *pcm,
        snd_pcm_hw_params_t *params, unsigned int val, int dir);
```

**⑦ 生效**

```c
int snd_pcm_hw_params(snd_pcm_t *pcm, snd_pcm_hw_params_t *params);
```

调用后**内部自动执行 `snd_pcm_prepare()`**，设备进入 `SND_PCM_STATE_PREPARED`（准备就绪）状态。

完整骨架：

```c
snd_pcm_hw_params_malloc(&hwparams);
snd_pcm_hw_params_any(pcm, hwparams);
snd_pcm_hw_params_set_access  (pcm, hwparams, SND_PCM_ACCESS_RW_INTERLEAVED);
snd_pcm_hw_params_set_format  (pcm, hwparams, SND_PCM_FORMAT_S16_LE);
snd_pcm_hw_params_set_rate    (pcm, hwparams, wav_fmt.SampleRate, 0);
snd_pcm_hw_params_set_channels(pcm, hwparams, wav_fmt.NumChannels);
snd_pcm_hw_params_set_period_size(pcm, hwparams, period_size, 0);
snd_pcm_hw_params_set_periods (pcm, hwparams, periods, 0);
ret = snd_pcm_hw_params(pcm, hwparams);
snd_pcm_hw_params_free(hwparams);
```

### 28.5.6 读写数据

```c
snd_pcm_sframes_t snd_pcm_writei(snd_pcm_t *pcm, const void *buffer, snd_pcm_uframes_t size);
snd_pcm_sframes_t snd_pcm_readi (snd_pcm_t *pcm, void *buffer, snd_pcm_uframes_t size);
```

- `buffer` 是**应用程序自己的缓冲区**，别和驱动层环形 buffer 搞混。
- `size` 单位是**帧**，通常一次一个周期。
- 成功返回实际读/写帧数；失败返回负错误码。
- **即使成功，返回帧数也可能小于 size**（被信号打断或 XRUN 时）。这时要把文件读位置往回挪 `(size - ret) * BlockAlign` 字节，否则会丢数据。

**阻塞 / 非阻塞**

| open 方式 | writei（buffer 满） | readi（buffer 空） |
|-----------|--------------------|-------------------|
| 阻塞（`mode=0`） | 阻塞等设备播完一个周期 | 阻塞等设备采满一个周期 |
| 非阻塞（`SND_PCM_NONBLOCK`） | 立即返回错误 | 立即返回错误 |

**非交错模式**要用 `snd_pcm_readn()` / `snd_pcm_writen()`。

### 28.5.7 示例 1：PCM 播放（同步）

原书例程：`28_alsa-lib/pcm_playback.c`。流程：解析 WAV 头 → 初始化 PCM → 循环"读一个周期 → 写一个周期"。

关键点：WAV 头用 `__attribute__((packed))` 保证结构体不被填充，否则读到的字段全错位。

```c
typedef struct WAV_RIFF {
    char ChunkID[4];        /* "RIFF" */
    uint32_t ChunkSize;     /* 从下一个地址开始到文件末尾的总字节数 */
    char Format[4];         /* "WAVE" */
} __attribute__ ((packed)) RIFF_t;

typedef struct WAV_FMT {
    char Subchunk1ID[4];    /* "fmt " */
    uint32_t Subchunk1Size; /* 16 for PCM */
    uint16_t AudioFormat;   /* PCM = 1 */
    uint16_t NumChannels;   /* 1=单声道 2=立体声 */
    uint32_t SampleRate;
    uint32_t ByteRate;      /* SampleRate*NumChannels*BitsPerSample/8 */
    uint16_t BlockAlign;    /* NumChannels*BitsPerSample/8 */
    uint16_t BitsPerSample;
} __attribute__ ((packed)) FMT_t;

typedef struct WAV_DATA {
    char Subchunk2ID[4];    /* "data" */
    uint32_t Subchunk2Size;
} __attribute__ ((packed)) DATA_t;
```

完整代码（本次已编译验证通过）：

```c
/***************************************************************
 文件名 : pcm_playback.c
 描述 : 一个简单地 PCM 播放示例代码--播放 WAV 音频文件（同步方式）
 ***************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <alsa/asoundlib.h>

#define PCM_PLAYBACK_DEV "hw:0,0"

typedef struct WAV_RIFF {
    char ChunkID[4];
    uint32_t ChunkSize;
    char Format[4];
} __attribute__ ((packed)) RIFF_t;

typedef struct WAV_FMT {
    char Subchunk1ID[4];
    uint32_t Subchunk1Size;
    uint16_t AudioFormat;
    uint16_t NumChannels;
    uint32_t SampleRate;
    uint32_t ByteRate;
    uint16_t BlockAlign;
    uint16_t BitsPerSample;
} __attribute__ ((packed)) FMT_t;

static FMT_t wav_fmt;

typedef struct WAV_DATA {
    char Subchunk2ID[4];
    uint32_t Subchunk2Size;
} __attribute__ ((packed)) DATA_t;

static snd_pcm_t *pcm = NULL;
static unsigned int buf_bytes;
static void *buf = NULL;
static int fd = -1;
static snd_pcm_uframes_t period_size = 1024;   /* 周期大小（帧） */
static unsigned int periods = 16;              /* 周期数 */

static int snd_pcm_init(void)
{
    snd_pcm_hw_params_t *hwparams = NULL;
    int ret;

    ret = snd_pcm_open(&pcm, PCM_PLAYBACK_DEV, SND_PCM_STREAM_PLAYBACK, 0);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_open error: %s: %s\n",
                PCM_PLAYBACK_DEV, snd_strerror(ret));
        return -1;
    }

    snd_pcm_hw_params_malloc(&hwparams);

    ret = snd_pcm_hw_params_any(pcm, hwparams);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_any error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params_set_access(pcm, hwparams, SND_PCM_ACCESS_RW_INTERLEAVED);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_set_access error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params_set_format(pcm, hwparams, SND_PCM_FORMAT_S16_LE);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_set_format error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params_set_rate(pcm, hwparams, wav_fmt.SampleRate, 0);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_set_rate error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params_set_channels(pcm, hwparams, wav_fmt.NumChannels);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_set_channels error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params_set_period_size(pcm, hwparams, period_size, 0);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_set_period_size error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params_set_periods(pcm, hwparams, periods, 0);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params_set_periods error: %s\n", snd_strerror(ret));
        goto err2;
    }

    ret = snd_pcm_hw_params(pcm, hwparams);
    snd_pcm_hw_params_free(hwparams);
    if (0 > ret) {
        fprintf(stderr, "snd_pcm_hw_params error: %s\n", snd_strerror(ret));
        goto err1;
    }

    buf_bytes = period_size * wav_fmt.BlockAlign;   /* 一个周期的字节数 */
    return 0;

err2:
    snd_pcm_hw_params_free(hwparams);
err1:
    snd_pcm_close(pcm);
    return -1;
}

static int open_wav_file(const char *file)
{
    RIFF_t wav_riff;
    DATA_t wav_data;
    int ret;

    fd = open(file, O_RDONLY);
    if (0 > fd) {
        fprintf(stderr, "open error: %s: %s\n", file, strerror(errno));
        return -1;
    }

    /* 读 RIFF chunk */
    ret = read(fd, &wav_riff, sizeof(RIFF_t));
    if (sizeof(RIFF_t) != ret ||
        strncmp("RIFF", wav_riff.ChunkID, 4) ||
        strncmp("WAVE", wav_riff.Format, 4)) {
        fprintf(stderr, "check error: %s\n", file);
        close(fd);
        return -1;
    }

    /* 读 sub-chunk-fmt */
    ret = read(fd, &wav_fmt, sizeof(FMT_t));
    if (sizeof(FMT_t) != ret || strncmp("fmt ", wav_fmt.Subchunk1ID, 4)) {
        fprintf(stderr, "check error: %s\n", file);
        close(fd);
        return -1;
    }

    printf("<<<<音频文件格式信息>>>>\n\n");
    printf(" file name    : %s\n", file);
    printf(" NumChannels  : %u\n", wav_fmt.NumChannels);
    printf(" SampleRate   : %u\n", wav_fmt.SampleRate);
    printf(" BlockAlign   : %u\n", wav_fmt.BlockAlign);
    printf(" BitsPerSample: %u\n\n", wav_fmt.BitsPerSample);

    /* 定位到 sub-chunk-data：RIFF头 + fmt头8字节 + fmt内容长度 */
    if (0 > lseek(fd, sizeof(RIFF_t) + 8 + wav_fmt.Subchunk1Size, SEEK_SET)) {
        perror("lseek error");
        close(fd);
        return -1;
    }

    /* 逐个 chunk 找 "data" */
    while (sizeof(DATA_t) == read(fd, &wav_data, sizeof(DATA_t))) {
        if (!strncmp("data", wav_data.Subchunk2ID, 4))
            return 0;   /* 文件位置正好在音频数据起点 */

        if (0 > lseek(fd, wav_data.Subchunk2Size, SEEK_CUR)) {
            perror("lseek error");
            close(fd);
            return -1;
        }
    }

    fprintf(stderr, "check error: %s\n", file);
    close(fd);
    return -1;
}

int main(int argc, char *argv[])
{
    int ret;
    int exit_code = EXIT_SUCCESS;

    if (2 != argc) {
        fprintf(stderr, "Usage: %s <audio_file>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    if (open_wav_file(argv[1]))
        exit(EXIT_FAILURE);

    if (snd_pcm_init())
        goto err1;

    buf = malloc(buf_bytes);
    if (NULL == buf) {
        perror("malloc error");
        goto err2;
    }

    for ( ; ; ) {
        ssize_t n;

        memset(buf, 0x00, buf_bytes);
        n = read(fd, buf, buf_bytes);
        if (0 > n) {
            perror("read error");
            exit_code = EXIT_FAILURE;
            break;
        }
        if (0 == n)
            break;                  /* 读完，正常结束 */

        ret = snd_pcm_writei(pcm, buf, period_size);
        if (0 > ret) {
            fprintf(stderr, "snd_pcm_writei error: %s\n", snd_strerror(ret));
            exit_code = EXIT_FAILURE;
            break;
        }
        else if (ret < (int)period_size) {
            /* 实写帧数不足：把文件读位置回退，下次重读这部分 */
            if (0 > lseek(fd, (ret - (long)period_size) * wav_fmt.BlockAlign, SEEK_CUR)) {
                perror("lseek error");
                exit_code = EXIT_FAILURE;
                break;
            }
        }
    }

    snd_pcm_drain(pcm);             /* 等 buffer 里剩的数据播完 */
    free(buf);
err2:
    snd_pcm_close(pcm);
err1:
    close(fd);

    printf("playback done, rc=%d\n", exit_code);
    return exit_code;
}
```

**为什么不用手动调 `snd_pcm_start()`？**

因为 `snd_pcm_writei/readi` 在设备处于 `PREPARED` 状态时，**内部会自动调 `snd_pcm_start()`**。
但要注意顺序：**必须先写入数据再启动**，否则启动瞬间 buffer 里一个周期数据都没有，直接 underrun 报错。

### 28.5.8 编译

```bash
. /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
cd ~/linux/IMX6ULL/app-test/28_alsa-lib

# 方式 A：直接用 SDK sysroot 里的 alsa-lib（推荐，最省事）
${CC} -o pcm_playback pcm_playback.c -lasound

# 方式 B：用自己移植的 alsa-lib（版本自己控制）
${CC} -o pcm_playback pcm_playback.c \
      -I$HOME/tools/alsa/include -L$HOME/tools/alsa/lib -lasound
```

两种都能过，产物 16608 字节 ARM 可执行文件，`readelf -d` 里 `NEEDED` 是 `libasound.so.2` + `libc.so.6`。

**不需要手动指定 `-I/-L`** 的原因：`${CC}` 里有 `--sysroot=...`，编译器自动把 `<sysroot>/usr/include` 加进头文件搜索路径、`<sysroot>/lib` 和 `<sysroot>/usr/lib` 加进库搜索路径。

![28-22 图28.5.5 cortexa7hf-neon-poky-lin.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-22%20%E5%9B%BE28.5.5%20cortexa7hf-neon-poky-lin.png)

![28-23 图28.5.6 alsa-lib 的头文件.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-23%20%E5%9B%BE28.5.6%20alsa-lib%20%E7%9A%84%E5%A4%B4%E6%96%87%E4%BB%B6.png)

![28-24 图28.5.7 alsa-lib 库文件.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-24%20%E5%9B%BE28.5.7%20alsa-lib%20%E5%BA%93%E6%96%87%E4%BB%B6.png)

![28-25 图28.5.8 CC 环境变量的内容.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-25%20%E5%9B%BE28.5.8%20CC%20%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E7%9A%84%E5%86%85%E5%AE%B9.png)

![28-26 图28.5.9 编译应用程序.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-26%20%E5%9B%BE28.5.9%20%E7%BC%96%E8%AF%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

### 28.5.9 部署和运行

```bash
# Ubuntu 端
scp pcm_playback test.wav root@192.168.77.148:/home/root/

# 板子端
cd /home/root

# 情况 1：用板子自带的 libasound（出厂系统 /usr/lib 里已有）
./pcm_playback test.wav

# 情况 2：用自己移植的 libasound（独立目录，不覆盖系统库）
export LD_LIBRARY_PATH=/home/root/alsa_lib/lib:$LD_LIBRARY_PATH
export ALSA_CONFIG_PATH=/home/root/alsa_lib/share/alsa/alsa.conf
./pcm_playback test.wav
```

成功会先打印 WAV 头信息，然后喇叭/耳机出声音。

![28-27 图28.5.10 将测试程序和 WAV 音频文件拷贝到开发板家目录.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-27%20%E5%9B%BE28.5.10%20%E5%B0%86%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%E5%92%8C%20WAV%20%E9%9F%B3%E9%A2%91%E6%96%87%E4%BB%B6%E6%8B%B7%E8%B4%9D%E5%88%B0%E5%BC%80%E5%8F%91%E6%9D%BF%E5%AE%B6%E7%9B%AE%E5%BD%95.png)

![28-29 图28.5.12 执行测试程序.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-29%20%E5%9B%BE28.5.12%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F.png)

### 28.5.10 示例 2：PCM 录音

原书例程：`28_alsa-lib/pcm_capture.c`。和播放的区别只有三处：`SND_PCM_STREAM_CAPTURE`、`snd_pcm_readi`、写文件。

```c
/* 1. 打开采集设备 */
snd_pcm_open(&pcm, "hw:0,0", SND_PCM_STREAM_CAPTURE, 0);

/* 2. 参数设置同上，采样率通常固定 44100（ES8388 只支持 44100，WM8960 可到 48000） */
snd_pcm_hw_params_set_rate(pcm, hwparams, 44100, 0);

/* 3. 循环读一个周期、写文件 */
while (!stop_flag) {
    ret = snd_pcm_readi(pcm, buf, period_size);
    if (-EPIPE == ret) {                 /* overrun，恢复后继续 */
        snd_pcm_prepare(pcm);
        continue;
    }
    else if (0 > ret) {
        fprintf(stderr, "snd_pcm_readi error: %s\n", snd_strerror(ret));
        break;
    }
    write(fd, buf, ret * 4);             /* 16bit 双声道，一帧 4 字节 */
}
```

> ⚠️ **原书写的是裸 PCM**（文件里没有 WAV 头），生成的 `cap.wav` 其实不是标准 wav，播放时要 `aplay -f cd -t raw` 之类指定格式。
> 本次实操版本改成**先写 44 字节 WAV 头占位、录完回填 `riff_size` / `data_size`**，这样生成的文件可以直接 `aplay` 播放。

WAV 头结构（44 字节）：

```c
struct wav_header {
    char     riff_id[4];        /* "RIFF" */
    uint32_t riff_size;         /* 文件总长 - 8 */
    char     wave_id[4];        /* "WAVE" */
    char     fmt_id[4];         /* "fmt " */
    uint32_t fmt_size;          /* 16 */
    uint16_t audio_format;      /* 1 = PCM */
    uint16_t channels;
    uint32_t sample_rate;
    uint32_t byte_rate;         /* sample_rate * channels * bits/8 */
    uint16_t block_align;       /* channels * bits/8 */
    uint16_t bits_per_sample;
    char     data_id[4];        /* "data" */
    uint32_t data_size;
} __attribute__ ((packed));
```

录音测试：

```bash
# 录音前先配好采集通道
amixer sset 'Right Input Mixer Boost' on
amixer sset 'Left Input Mixer Boost' on
amixer sset 'ADC' 255
amixer sset 'Capture' 63

./pcm_capture cap.wav        # Ctrl+C 停止
aplay cap.wav                # 回放验证
```

![28-30 图28.5.13 编译示例代码.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-30%20%E5%9B%BE28.5.13%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

![28-31 图28.5.14 声卡配置（录音） 左右声道的 Mixer Boo.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-31%20%E5%9B%BE28.5.14%20%E5%A3%B0%E5%8D%A1%E9%85%8D%E7%BD%AE%EF%BC%88%E5%BD%95%E9%9F%B3%EF%BC%89%20%E5%B7%A6%E5%8F%B3%E5%A3%B0%E9%81%93%E7%9A%84%20Mixer%20Boo.png)

![28-32 图28.5.15 录音 执行测试程序之后就开始录音了接着.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-32%20%E5%9B%BE28.5.15%20%E5%BD%95%E9%9F%B3%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%E4%B9%8B%E5%90%8E%E5%B0%B1%E5%BC%80%E5%A7%8B%E5%BD%95%E9%9F%B3%E4%BA%86%E6%8E%A5%E7%9D%80.png)

**两种录音输入源**

| 输入源 | 说明 | 参考图 |
|--------|------|--------|
| 板载麦克风 | 直接对着板子说话即可 | 图 28.5.16 |
| LINE_IN | 手机/电脑耳机口通过音频线输入 | 图 28.5.19 |

![28-33 图28.5.16 板载麦克风.jpg](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-33%20%E5%9B%BE28.5.16%20%E6%9D%BF%E8%BD%BD%E9%BA%A6%E5%85%8B%E9%A3%8E.jpg)

![28-34 图28.5.17 生成 cap.wav 文件 生成的文件是一个纯音.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-34%20%E5%9B%BE28.5.17%20%E7%94%9F%E6%88%90%20cap.wav%20%E6%96%87%E4%BB%B6%20%E7%94%9F%E6%88%90%E7%9A%84%E6%96%87%E4%BB%B6%E6%98%AF%E4%B8%80%E4%B8%AA%E7%BA%AF%E9%9F%B3.png)

![28-35 图28.5.18 使用 aplay 播放录制的音频.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-35%20%E5%9B%BE28.5.18%20%E4%BD%BF%E7%94%A8%20aplay%20%E6%92%AD%E6%94%BE%E5%BD%95%E5%88%B6%E7%9A%84%E9%9F%B3%E9%A2%91.png)

![28-36 图28.5.19 LINE_IN 接口.jpg](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-36%20%E5%9B%BE28.5.19%20LINE_IN%20%E6%8E%A5%E5%8F%A3.jpg)

使用 LINE_IN 前要切输入通道：

```bash
amixer cset name='ZC Timeout Switch' 'on'
amixer cset name='Left PGA Mux' 'DifferentialL'
amixer cset name='Right PGA Mux' 'DifferentialR'
amixer cset name='Left Mixer Left Bypass Switch' 'on'
amixer cset name='Right Mixer Right Bypass Switch' 'on'
amixer cset name='Left Mixer Left Playback Switch' 'on'
amixer cset name='Right Mixer Right Playback Switch' 'on'
amixer cset name='Differential Mux' 'Line 1'
amixer cset name='Left Line Mux' 'Line 1L'
amixer cset name='Right Line Mux' 'Line 1R'
```

---

## 28.6 使用异步方式

前两个例程都是**同步**读写：`writei/readi` 一阻塞，程序什么都干不了。

异步方式只需要**注册一个异步处理函数**，库内部帮你做了三件事：使能异步 I/O、设置属主、注册信号处理（SIGIO）。

### 28.6.1 两个 API

```c
int snd_async_add_pcm_handler(snd_async_handler_t **handler,
                              snd_pcm_t *pcm,
                              snd_async_callback_t callback,
                              void *private_data);
```

| 参数 | 说明 |
|------|------|
| `handler` | 返回异步处理对象句柄 |
| `pcm` | PCM 设备句柄 |
| `callback` | 回调函数，原型 `typedef void (*snd_async_callback_t)(snd_async_handler_t *handler)` |
| `private_data` | 私有数据，回调里用 `snd_async_handler_get_callback_private(handler)` 取 |

回调里拿 PCM 句柄：

```c
snd_pcm_t *pcm_handle = snd_async_handler_get_pcm(handler);
```

```c
snd_pcm_sframes_t snd_pcm_avail_update(snd_pcm_t *pcm);
```

- **播放**：返回环形缓冲区里**当前还能写多少帧**
- **录音**：返回**当前可读多少帧**

### 28.6.2 触发时机

- **播放**：驱动每播完一个周期就产生一个空闲周期 → 发 SIGIO → 应用跳到回调补充数据。通常是**先把 buffer 填满**，之后靠回调持续喂。
- **录音**：驱动采集满一个周期写入 buffer → 发 SIGIO → 应用跳到回调读走。

### 28.6.3 异步播放核心代码

```c
static void snd_playback_async_callback(snd_async_handler_t *handler)
{
    snd_pcm_t *handle = snd_async_handler_get_pcm(handler);
    snd_pcm_sframes_t avail;
    int ret;

    avail = snd_pcm_avail_update(handle);       /* 还能写多少帧 */
    while (avail >= (snd_pcm_sframes_t)period_size) {
        memset(buf, 0x00, buf_bytes);
        ret = read(fd, buf, buf_bytes);
        if (0 >= ret)
            goto out;                            /* 文件读完 */

        ret = snd_pcm_writei(handle, buf, period_size);
        if (0 > ret) {
            fprintf(stderr, "snd_pcm_writei error: %s\n", snd_strerror(ret));
            goto out;
        }
        else if (ret < (int)period_size) {
            if (0 > lseek(fd, (ret - (long)period_size) * wav_fmt.BlockAlign, SEEK_CUR))
                goto out;
        }

        avail = snd_pcm_avail_update(handle);    /* 更新剩余空间 */
    }
    return;

out:
    snd_pcm_drain(handle);
    snd_pcm_close(handle);
    free(buf);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

注册：

```c
snd_async_handler_t *async_handler = NULL;
ret = snd_async_add_pcm_handler(&async_handler, pcm, snd_playback_async_callback, NULL);
if (0 > ret)
    fprintf(stderr, "snd_async_add_pcm_handler error: %s\n", snd_strerror(ret));
```

主线程把 buffer 填满后就可以空转等待：

```c
for ( ; ; )
    pause();        /* 被 SIGIO 打断时返回，继续等 */
```

![28-37 图28.6.1 编译示例代码.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-37%20%E5%9B%BE28.6.1%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

---

## 28.7 使用 poll() 函数

用 I/O 多路复用（`poll`）实现，思路和第 13 章的高级 I/O 一样。

### 28.7.1 三步

**① 取轮询描述符数量**

```c
int snd_pcm_poll_descriptors_count(snd_pcm_t *pcm);
```

**② 分配 `struct pollfd` 数组并用 `snd_pcm_poll_descriptors()` 填充**

```c
int snd_pcm_poll_descriptors(snd_pcm_t *pcm, struct pollfd *pfds, unsigned int space);
```

```c
struct pollfd *pfds = NULL;
int count = snd_pcm_poll_descriptors_count(pcm);
pfds = calloc(count, sizeof(struct pollfd));
snd_pcm_poll_descriptors(pcm, pfds, count);
```

**③ poll + `snd_pcm_poll_descriptors_revents()` 解释事件**

```c
int snd_pcm_poll_descriptors_revents(snd_pcm_t *pcm, struct pollfd *pfds,
                                     unsigned int nfds, unsigned short *revents);
```

> ⚠️ **不要直接读 `pfds[i].revents`**。必须调 `snd_pcm_poll_descriptors_revents()`，因为它会对 poll 返回的 revents 掩码做"分解"来纠正语义（POLLIN = 可读，POLLOUT = 可写）。

### 28.7.2 播放循环

```c
/* 先塞一个周期把设备跑起来 */
feed_one_period();

while (!eof_flag) {
    ret = poll(pfds, count, -1);
    if (0 > ret) { perror("poll error"); break; }

    ret = snd_pcm_poll_descriptors_revents(pcm, pfds, count, &revents);
    if (0 > ret) break;

    if (revents & POLLERR)  break;                 /* I/O 错误 */
    if (revents & POLLOUT)  feed_one_period();     /* 可写，补一个周期 */
    /* 录音方向则判 POLLIN，然后 snd_pcm_readi */
}
```

---

## 28.8 PCM 设备的状态

```c
snd_pcm_state_t snd_pcm_state(snd_pcm_t *pcm);
```

```c
enum snd_pcm_state_t {
    SND_PCM_STATE_OPEN = 0,
    SND_PCM_STATE_SETUP,
    SND_PCM_STATE_PREPARED,
    SND_PCM_STATE_RUNNING,
    SND_PCM_STATE_XRUN,
    SND_PCM_STATE_DRAINING,
    SND_PCM_STATE_PAUSED,
    SND_PCM_STATE_SUSPENDED,
    SND_PCM_STATE_DISCONNECTED,
    SND_PCM_STATE_LAST = SND_PCM_STATE_DISCONNECTED,
    SND_PCM_STATE_PRIVATE1 = 1024
};
```

| 状态 | 含义 | 怎么进来 / 怎么出去 |
|------|------|--------------------|
| `OPEN` | 已打开 | `snd_pcm_open()` 之后 |
| `SETUP` | 已初始化、参数配好 | `hw_params` 中间态；`drop()/drain()` 之后回到这里 |
| `PREPARED` | 准备好了，可以开始播/录 | `snd_pcm_hw_params()` 或 `snd_pcm_prepare()` 之后 |
| `RUNNING` | 正在运行（播/录） | `snd_pcm_start()`，或 `readi/writei` 内部自动启动 |
| `XRUN` | 发生 over/underrun | 用 `snd_pcm_prepare()` 恢复 |
| `DRAINING` | 排空中（播放中或采集已停） | — |
| `PAUSED` | 暂停 | `snd_pcm_pause(pcm, 1)` 暂停，`0` 恢复 |
| `SUSPENDED` | 硬件挂起 | `snd_pcm_resume()` 恢复（不丢样本） |
| `DISCONNECTED` | 硬件断开 | — |

**关键状态函数**

```c
int snd_pcm_prepare(snd_pcm_t *pcm);   /* → PREPARED */
int snd_pcm_start  (snd_pcm_t *pcm);   /* → RUNNING  */
int snd_pcm_drop   (snd_pcm_t *pcm);   /* 立即停，丢弃挂起帧 → SETUP */
int snd_pcm_drain  (snd_pcm_t *pcm);   /* 处理完挂起帧再停 → SETUP */
int snd_pcm_pause  (snd_pcm_t *pcm, int enable);  /* 1=暂停 0=恢复 */
int snd_pcm_resume (snd_pcm_t *pcm);
```

**drop 和 drain 的区别**（很重要）：

- **`drop`**：立刻停，缓冲区里没播完的帧直接丢掉。
- **`drain`**：等缓冲区里待播的帧全部播完再停；录音时则先检索残留帧。

**硬件能力检查**

```c
int snd_pcm_hw_params_can_pause (const snd_pcm_hw_params_t *params);  /* 1=支持 */
int snd_pcm_hw_params_can_resume(const snd_pcm_hw_params_t *params);  /* 1=支持 */
```

不是所有硬件都支持暂停/挂起恢复，调之前先判断。

![28-38 图28.8.1 PCM 设备状态之间的转换关系示意.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-38%20%E5%9B%BE28.8.1%20PCM%20%E8%AE%BE%E5%A4%87%E7%8A%B6%E6%80%81%E4%B9%8B%E9%97%B4%E7%9A%84%E8%BD%AC%E6%8D%A2%E5%85%B3%E7%B3%BB%E7%A4%BA%E6%84%8F.png)

![28-39 图28.8.2 运行测试程序 运行之后开始播放音乐此时.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-39%20%E5%9B%BE28.8.2%20%E8%BF%90%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%20%E8%BF%90%E8%A1%8C%E4%B9%8B%E5%90%8E%E5%BC%80%E5%A7%8B%E6%92%AD%E6%94%BE%E9%9F%B3%E4%B9%90%E6%AD%A4%E6%97%B6.png)

![28-40 图28.8.3 snd_pcm_writei 函数的错误返回值描.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-40%20%E5%9B%BE28.8.3%20snd_pcm_writei%20%E5%87%BD%E6%95%B0%E7%9A%84%E9%94%99%E8%AF%AF%E8%BF%94%E5%9B%9E%E5%80%BC%E6%8F%8F.png)

---

## 28.9 混音器设置

`alsamixer` / `amixer` 本身就是基于这几个 API 写的。

### 28.9.1 打开混音器（4 步）

```c
int snd_mixer_open(snd_mixer_t **mixerp, int mode);                        /* ① 打开空混音器 */
int snd_mixer_attach(snd_mixer_t *mixer, const char *name);                /* ② 关联声卡控制设备 */
int snd_mixer_selem_register(snd_mixer_t *mixer,
        struct snd_mixer_selem_regopt *options, snd_mixer_class_t **classp); /* ③ 注册，参数传 NULL */
int snd_mixer_load(snd_mixer_t *mixer);                                    /* ④ 加载 */
```

```c
snd_mixer_t *mixer = NULL;
snd_mixer_open(&mixer, 0);
snd_mixer_attach(mixer, "hw:0");        /* "hw:i" → /dev/snd/controlC0 */
snd_mixer_selem_register(mixer, NULL, NULL);
snd_mixer_load(mixer);
```

![28-41 图28.9.1 Mixer Interface 模块.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-41%20%E5%9B%BE28.9.1%20Mixer%20Interface%20%E6%A8%A1%E5%9D%97.png)

### 28.9.2 遍历元素找配置项

alsa-lib 把每个配置项叫**元素（element）**，用 `snd_mixer_elem_t` 描述。

```c
snd_mixer_elem_t *snd_mixer_first_elem(snd_mixer_t *mixer);
snd_mixer_elem_t *snd_mixer_last_elem (snd_mixer_t *mixer);
snd_mixer_elem_t *snd_mixer_elem_next (snd_mixer_elem_t *elem);
snd_mixer_elem_t *snd_mixer_elem_prev (snd_mixer_elem_t *elem);
const char       *snd_mixer_selem_get_name(snd_mixer_elem_t *elem);
```

```c
snd_mixer_elem_t *elem = snd_mixer_first_elem(mixer);
while (elem) {
    if (!strcmp(snd_mixer_selem_get_name(elem), "Headphone")) {
        /* 找到了 */
    }
    elem = snd_mixer_elem_next(elem);
}
```

### 28.9.3 判断元素类型

配置值就两种：**volume 型**（范围数值，如音量）和 **switch 型**（bool，开关）。

```c
int snd_mixer_selem_has_playback_volume(snd_mixer_elem_t *elem);  /* 1=是volume */
int snd_mixer_selem_has_capture_volume (snd_mixer_elem_t *elem);

int snd_mixer_selem_has_playback_switch(snd_mixer_elem_t *elem);  /* 1=是switch */
int snd_mixer_selem_has_capture_switch (snd_mixer_elem_t *elem);

int snd_mixer_selem_is_playback_mono(snd_mixer_elem_t *elem);
int snd_mixer_selem_is_capture_mono (snd_mixer_elem_t *elem);
```

通道枚举：

```c
enum snd_mixer_selem_channel_id_t {
    SND_MIXER_SCHN_UNKNOWN = -1,
    SND_MIXER_SCHN_FRONT_LEFT = 0,   /* 左前 —— 双声道元素通常只有这个和右前 */
    SND_MIXER_SCHN_FRONT_RIGHT,      /* 右前 */
    SND_MIXER_SCHN_REAR_LEFT,
    SND_MIXER_SCHN_REAR_RIGHT,
    SND_MIXER_SCHN_FRONT_CENTER,
    SND_MIXER_SCHN_WOOFER,
    SND_MIXER_SCHN_SIDE_LEFT,
    SND_MIXER_SCHN_SIDE_RIGHT,
    SND_MIXER_SCHN_REAR_CENTER,
    SND_MIXER_SCHN_LAST = 31,
    SND_MIXER_SCHN_MONO = SND_MIXER_SCHN_FRONT_LEFT   /* 单声道 */
};
```

### 28.9.4 读写音量

```c
int snd_mixer_selem_get_playback_volume(snd_mixer_elem_t *elem,
        snd_mixer_selem_channel_id_t channel, long *value);
int snd_mixer_selem_get_capture_volume (snd_mixer_elem_t *elem,
        snd_mixer_selem_channel_id_t channel, long *value);

int snd_mixer_selem_set_playback_volume(snd_mixer_elem_t *elem,
        snd_mixer_selem_channel_id_t channel, long value);
int snd_mixer_selem_set_capture_volume (snd_mixer_elem_t *elem,
        snd_mixer_selem_channel_id_t channel, long value);

/* 一次性设置所有声道 */
int snd_mixer_selem_set_playback_volume_all(snd_mixer_elem_t *elem, long value);
int snd_mixer_selem_set_capture_volume_all (snd_mixer_elem_t *elem, long value);

/* 取范围 */
int snd_mixer_selem_get_playback_volume_range(snd_mixer_elem_t *elem,
        long *min, long *max);
int snd_mixer_selem_get_capture_volume_range (snd_mixer_elem_t *elem,
        long *min, long *max);
```

例：

```c
long value;
snd_mixer_selem_get_playback_volume(elem, SND_MIXER_SCHN_FRONT_LEFT, &value);
snd_mixer_selem_set_playback_volume_all(elem, 105);   /* 左右都设 105 */
```

### 28.9.5 完整流程

```c
static int open_mixer(void)
{
    int ret;

    ret = snd_mixer_open(&mixer, 0);
    if (0 > ret) { fprintf(stderr, "snd_mixer_open error: %s\n", snd_strerror(ret)); return -1; }

    ret = snd_mixer_attach(mixer, "hw:0");
    if (0 > ret) { fprintf(stderr, "snd_mixer_attach error: %s\n", snd_strerror(ret)); return -1; }

    ret = snd_mixer_selem_register(mixer, NULL, NULL);
    if (0 > ret) { fprintf(stderr, "snd_mixer_selem_register error: %s\n", snd_strerror(ret)); return -1; }

    ret = snd_mixer_load(mixer);
    if (0 > ret) { fprintf(stderr, "snd_mixer_load error: %s\n", snd_strerror(ret)); return -1; }

    playback_vol_elem = mixer_find_elem(mixer, "Headphone");   /* 见上面的遍历函数 */
    if (NULL == playback_vol_elem) {
        fprintf(stderr, "can't find mixer element \"Headphone\"\n");
        return -1;
    }
    return 0;
}
```

然后把"异步播放 + 混音器设音量"合起来：`pcm_playback_mixer.c` 就是这个思路，播放前先把 `Headphone` 音量设到 105，再走异步回调喂数据。

![28-42 图28.9.2 编译示例代码.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-42%20%E5%9B%BE28.9.2%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

![28-43 图28.9.3 执行测试程序 命令执行之后会显示操作方式.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-43%20%E5%9B%BE28.9.3%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%20%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E4%B9%8B%E5%90%8E%E4%BC%9A%E6%98%BE%E7%A4%BA%E6%93%8D%E4%BD%9C%E6%96%B9%E5%BC%8F.png)

---

## 28.10 回环测试

alsa-utils 自带 **alsaloop**，能边录边放。

```bash
alsaloop --help
alsaloop -t 1000
```

原理看着简单（录完直接播），实际坑不少：**录音是 ADC 过程、播放是 DAC 过程，硬件处理一个周期花的时间并不相同**，所以很容易 XRUN。怎么把周期和线程节奏设计好，才是关键。

想研究实现就看 alsa-utils 源码里的 `alsaloop/`（aplay、alsamixer、amixer、alsactl 的源码也都在同一个包里）。

![28-44 图28.10.1 alsaloop 工具.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-44%20%E5%9B%BE28.10.1%20alsaloop%20%E5%B7%A5%E5%85%B7.png)

![28-45 图28.10.2 alsaloop 源码 除了 alsaloop.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-45%20%E5%9B%BE28.10.2%20alsaloop%20%E6%BA%90%E7%A0%81%20%E9%99%A4%E4%BA%86%20alsaloop.png)

---

## 28.11 总结：ALSA 插件（plugin）

调 `snd_pcm_open()` 时填的 `"hw:0,0"` 里的 **`hw` 其实是一个插件名**，冒号后两个数字是传给插件的参数（卡号、设备号）。

### 配置文件加载链

```
snd_pcm_open()
   └─> /usr/share/alsa/alsa.conf          主入口，定义通用插件
         ├─> /etc/asound.conf             系统级用户自定义
         └─> ~/.asoundrc                  用户级用户自定义
```

出厂系统有 `/etc/asound.conf`，没有 `~/.asoundrc`。

![28-46 图28.11.1 alsa.conf 文件.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-46%20%E5%9B%BE28.11.1%20alsa.conf%20%E6%96%87%E4%BB%B6.png)

`/etc/asound.conf` 里每个 `pcm.名字 { ... }` 就是一个插件定义，`type` 字段决定插件类型：

![28-47 图28.11.2 etcasound.conf 文件中定义的插件.png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%85%AB%E7%AB%A0%20ALSA%E9%9F%B3%E9%A2%91%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/28-47%20%E5%9B%BE28.11.2%20etcasound.conf%20%E6%96%87%E4%BB%B6%E4%B8%AD%E5%AE%9A%E4%B9%89%E7%9A%84%E6%8F%92%E4%BB%B6.png)

| 插件 | 作用 |
|------|------|
| **hw** | 直接和内核 ALSA 驱动通信，**没有任何转换**，完全原始。配不支持的参数会直接失败 |
| **plughw** | 在 hw 之上加了软件特性（采样率转换、格式转换）。比如硬件不支持 48000，用 plughw 就能播 |
| **dmix** | 混音，多个应用的音频数据混合输出 |
| **softvol** | 软件音量 |
| 还有 | `mmap_emul`、`shm`、`linear`、`plug`、`multi`、`share`、`dsnoop` |

**实用结论**：如果 `set_rate()` 报错说设备不支持某个采样率，把设备名从 `"hw:0,0"` 换成 `"plughw:0,0"`（或 `"default"`）通常就能解决。

参考文档：
- `https://www.alsa-project.org/main/index.php/Asoundrc`
- `https://www.alsa-project.org/alsa-doc/alsa-lib/pcm_plugins.html`

---

## 附：本次实操记录（真机环境）

### 环境

| 项 | 值 |
|----|-----|
| Ubuntu | 192.168.250.163 |
| 板子 | ATK-IMX6U / 192.168.77.148，ALPHA（WM8960） |
| 工具链 | `/opt/fsl-imx-x11/4.1.15-2.1.0`（`arm-poky-linux-gnueabi-gcc 5.3.0`） |
| 板子声卡 | card0 = `wm8960-audio` |
| 板子 `/dev/snd` | `controlC0`、`pcmC0D0c`、`pcmC0D0p`、`pcmC0D1c`、`pcmC0D1p`、`timer` 全在 |
| 板子自带工具 | `aplay`、`arecord`、`amixer`、`alsamixer` 都在 `/usr/bin` |

### 已完成

| 步骤 | 结果 |
|------|------|
| 下载 alsa-lib-1.1.5 | 957K，`bzip2 -t` 校验通过 |
| 交叉 configure | `cfg_rc=0`，CC 带 `--sysroot` |
| make / make install | `make_rc=0`、`install_rc=0` → `~/tools/alsa` |
| strip | 3.8MB → **840K** |
| 部署板子 | `/home/root/alsa_lib/lib`（`libasound.so.2.0.0` 859212 字节）+ `share/alsa` |
| 4 个例程编译 | **全部 rc=0**（详见下表） |
| 上传板子 | `/home/root/pcm_*` 4 个可执行文件 |
| 测试 WAV | `/home/root/test_alsa.wav`（3 秒 440Hz 正弦，16bit 44.1k 双声道） |

**编译产物**

| 程序 | 大小 | 说明 |
|------|------|------|
| `pcm_playback` | 16608 | 同步播放 WAV |
| `pcm_capture` | 23580 | 录音存 WAV（带 44 字节头） |
| `pcm_playback_poll` | 17644 | poll 方式播放 |
| `pcm_playback_mixer` | 19784 | 异步播放 + 混音器设音量 |

两条编译路线都验证过：

```bash
# A. SDK sysroot 自带 alsa-lib
${CC} -o pcm_playback pcm_playback.c -lasound

# B. 用自己移植的 alsa-lib
${CC} -o pcm_playback pcm_playback.c \
      -I$HOME/tools/alsa/include -L$HOME/tools/alsa/lib -lasound
```

`readelf -d` 确认 `NEEDED = libasound.so.2 + libc.so.6`。

### 运行命令（本次未执行，留给你跑）

```bash
# 上板
scp pcm_playback pcm_capture pcm_playback_poll pcm_playback_mixer test_alsa.wav \
    root@192.168.77.148:/home/root/
```

```bash
# 板子上执行
cd /home/root

# 用板子自带 libasound（最省事）
./pcm_playback test_alsa.wav
./pcm_playback_poll test_alsa.wav
./pcm_playback_mixer test_alsa.wav
./pcm_capture cap.wav          # Ctrl+C 停止

# 用自己移植的 libasound
export LD_LIBRARY_PATH=/home/root/alsa_lib/lib:$LD_LIBRARY_PATH
export ALSA_CONFIG_PATH=/home/root/alsa_lib/share/alsa/alsa.conf
./pcm_playback test_alsa.wav
```

### 踩坑清单

1. **alsa-lib 源码包下载不完整**：第一次只拿到 592K，`tar -xjf` 报 `Compressed file ends unexpectedly`。必须 `bzip2 -t` 校验，完整包 **957K**。
2. **`share/alsa` 部署路径**：打包时 `tar -czf xxx -C share alsa` 生成的顶层是 `alsa/`，解到 `alsa_lib/` 下会变成 `alsa_lib/alsa/`，要手动挪到 `alsa_lib/share/alsa/`。
3. **`alsa.conf` 不能漏**：用自己移植的库时必须设 `ALSA_CONFIG_PATH`，否则打开 PCM 直接失败。
4. **`pcm_capture.c` 原书是裸 PCM**：录出来的文件没有 WAV 头，`aplay` 播不了。本次版本改成写 44 字节头 + 录完回填长度。
5. **ES8388 只支持 44100**：WM8960 能到 48000，换板子记得改采样率，否则 `set_rate` 报错。
6. **采样率不支持时换 `plughw:0,0`**：`hw` 插件不做任何转换，硬件不支持就直接失败；`plughw` 有软件重采样。
7. **`poll` 后不要读 `pfds[i].revents`**：必须用 `snd_pcm_poll_descriptors_revents()`，否则事件语义可能反了。
8. **`screen_base` 那种老坑这次没有**：ALSA 例程不涉及 mmap 显存，四个程序都不会段错误。
9. **Mini 板不能跑**：没有音频芯片，`/dev/snd` 是空的。
