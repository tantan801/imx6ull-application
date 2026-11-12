# 提高篇第二十章 在 LCD 上显示 jpeg 图像

> 配套例程：`20_libjpeg/show_jpeg_image.c`
> 核心：把 `.jpg/.jpeg` 用 libjpeg 解压成 RGB，再转 RGB565 刷进 framebuffer 显示。
> 前置：第 19 章 FrameBuffer（`/dev/fb0`、`FBIOGET_VSCREENINFO`、`mmap`）已经会了，本章只是把"数据来源"从 BMP 文件换成 libjpeg 解码结果。

常用图片格式三种：**JPEG（JPG）、PNG、BMP**。

| 格式 | 压缩 | 特点 |
|------|------|------|
| BMP | 无压缩 | 无失真、解析简单，但文件极大，不适合存磁盘 |
| JPEG | 有损 | 压缩比高，适合存储，本章主题 |
| PNG | 无损 | 压缩比高、体积小、支持透明，下一章主题 |

---

## 20.1 JPEG 简介

- **JPEG** = Joint Photographic Experts Group，国际标准组织为**静态图像**建立的第一个国际数字图像压缩标准，至今应用最广。
- 采用**有损压缩**，压缩比远超传统压缩算法。
- 丢掉的"损失"是人眼不容易察觉的部分：它利用了**人眼对高频色彩信息不敏感**的特点，大幅压缩数据。
- 文件后缀通常 `.jpg` 或 `.jpeg`。

一句话：JPEG 是"用一点人眼看不出的画质换体积"的压缩标准，显示前必须先解压成 RGB 原始数据。

---

## 20.2 libjpeg 简介

- **libjpeg** 是一个完全用 **C 语言**编写的开源函数库，包含：
  - **JPEG 解码**（解压缩，jpg → RGB）
  - **JPEG 编码**（压缩，RGB → jpg）
  - 其他 JPEG 功能
- 想在 LCD 上显示 jpg，就得先解压拿到 RGB 数据；解压算法复杂，直接用现成库即可。
- 源码下载：<http://www.ijg.org/files/>
- 本章只讲**解码**，编码不讲。

---

## 20.3 libjpeg 移植

### 20.3.1 下载源码包

打开 <http://www.ijg.org/files/>：

![20-1 图20.3.1 libjpeg 下载链接页面](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-1%20%E5%9B%BE20.3.1%20libjpeg%20%E4%B8%8B%E8%BD%BD%E9%93%BE%E6%8E%A5%E9%A1%B5%E9%9D%A2%20%E7%9B%AE%E5%89%8D%E6%9C%80%E6%96%B0%E7%9A%84%E4%B8%80%E4%B8%AA%E7%89%88%E6%9C%AC.png)

- 最新版 v9d（2020 年）；**书里用 v9b**，文件名 `jpegsrc.v9b.tar.gz`。
- 出厂系统其实已经移植过 libjpeg，但**版本太旧**，所以自己重新移植。

![20-2 图20.3.2 jpegsrc.v9b.tar.gz 源码包](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-2%20%E5%9B%BE20.3.2%20jpegsrc.v9b.tar.gz%20%E6%BA%90%E7%A0%81%E5%8C%85.png)

### 20.3.2 编译源码

把 `jpegsrc.v9b.tar.gz` 拷到 Ubuntu 家目录，解压：

```bash
tar -xzf jpegsrc.v9b.tar.gz
```

得到源码目录 `jpeg-9b`，里面就是 libjpeg 源码：

![20-5 图20.3.6 libjpeg 源码目录下的文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-5%20%E5%9B%BE20.3.6%20libjpeg%20%E6%BA%90%E7%A0%81%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6.png)

![20-3 图20.3.4 将 jpegsrc.v9b.tar.gz 文件解压](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-3%20%E5%9B%BE20.3.4%20%E5%B0%86%20jpegsrc.v9b.tar.gz%20%E6%96%87%E4%BB%B6%E8%A7%A3.png)

在 `~/tools` 下建安装目录 `jpeg`（和 tslib 一样）：

![20-4 图20.3.5 创建 jpeg 文件夹](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-4%20%E5%9B%BE20.3.5%20%E5%88%9B%E5%BB%BA%20jpeg%20%E6%96%87%E4%BB%B6%E5%A4%B9%20%E5%9B%9E%E5%88%B0%E5%AE%B6%E7%9B%AE%E5%BD%95%E4%B8%8B%E8%BF%9B%E5%85%A5%E5%88%B0%20li.png)

```bash
mkdir -p ~/tools/jpeg
```

进入源码目录，三步走：**配置 → 编译 → 安装**。先初始化交叉编译环境（已初始化可跳过）：

```bash
source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
```

配置工程：

```bash
./configure --host=arm-poky-linux-gnueabi --prefix=/home/dt/tools/jpeg/
```

- `--host`：指定交叉编译产物运行的平台，**填交叉编译器名称前缀**（如 `arm-poky-linux-gnueabi-gcc` 的前缀就是 `arm-poky-linux-gnueabi`）。
- `--prefix`：库的**安装路径**。
- 可执行 `./configure --help` 查看全部选项。

> **换成你自己的工具链时**：如果用的是正点原子较新的 `arm-linux-gnueabihf-gcc`，就把 `--host` 改成 `arm-linux-gnueabihf`、`--prefix` 改成 `$HOME/tools/jpeg`。二者等价，只是前缀不同。

![20-6 图20.3.7 配置工程](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-6%20%E5%9B%BE20.3.7%20%E9%85%8D%E7%BD%AE%E5%B7%A5%E7%A8%8B%20%E6%8E%A5%E7%9D%80%E6%89%A7%E8%A1%8C%20make%20%E5%91%BD%E4%BB%A4%E7%BC%96%E8%AF%91%E5%B7%A5%E7%A8%8B.png)

编译 + 安装：

```bash
make
make install
```

![20-7 图20.3.8 编译工程](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-7%20%E5%9B%BE20.3.8%20%E7%BC%96%E8%AF%91%E5%B7%A5%E7%A8%8B%20%E7%BC%96%E8%AF%91%E5%AE%8C%E6%88%90%E4%B9%8B%E5%90%8E%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4%E5%AE%89%E8%A3%85%20libjpe.png)

![20-8 图20.3.9 安装](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-8%20%E5%9B%BE20.3.9%20%E5%AE%89%E8%A3%85%20%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E5%AE%8C%E6%88%90%E4%B9%8B%E5%90%8E.png)

### 20.3.3 安装目录说明

进入 `~/tools/jpeg`，结构基本同 tslib（只是没有 `etc`）：

```
tools/jpeg/
├── bin/       # 测试工具（djpeg、cjpeg 等）
├── include/   # 4 个头文件，应用里只 include jpeglib.h
└── lib/       # 动态库
```

![20-9 图20.3.10 安装目录下的文件夹](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-9%20%E5%9B%BE20.3.10%20%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%E5%A4%B9%20%E4%B8%8E%20tslib%20%E5%BA%93%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%E4%B8%8B.png)

![20-10 图20.3.11 include 头文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-10%20%E5%9B%BE20.3.11%20%E5%A4%B4%E6%96%87%E4%BB%B6%20%E5%9C%A8.png)

`lib` 目录里：

![20-11 图20.3.12 库文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-11%20%E5%9B%BE20.3.12%20%E5%BA%93%E6%96%87%E4%BB%B6.png)

- `libjpeg.so`、`libjpeg.so.9` 都是**符号链接**，最终指向 `libjpeg.so.9.2.0`。

### 20.3.4 移植到开发板

出厂系统已有旧版 libjpeg，这里不用它，用自己编的。做法：

1. `bin/` 下所有测试工具 → 板子 `/usr/bin`
2. `lib/` 下所有库文件 → 板子 `/usr/lib`
3. **注意符号链接不能破坏** → 整目录打包再传：

```bash
# Ubuntu 端：在 lib 目录下打包（保留符号链接）
tar -czf lib.tar.gz ./*

# 板子端：先删出厂旧库，再解压到 /usr/lib
rm -rf /usr/lib/libjpeg.*
tar -xzf lib.tar.gz -C /usr/lib
```

![20-12 图20.3.13 将 lib 目录打包](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-12%20%E5%9B%BE20.3.13%20%E5%B0%86%20lib%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%89%80%E6%9C%89%E6%96%87%E4%BB%B6%E6%89%93%E5%8C%85%20%E5%86%8D%E5%B0%86%20lib..png)

![20-13 图20.3.14 删除出厂系统原有的 libjpeg 库](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-13%20%E5%9B%BE20.3.14%20%E5%88%A0%E9%99%A4%E5%87%BA%E5%8E%82%E7%B3%BB%E7%BB%9F%E5%8E%9F%E6%9C%89%E7%9A%84%20libjpeg%20%E5%BA%93.png)

![20-14 图20.3.15 将 lib.tar.gz 解压到 /usr/lib](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-14%20%E5%9B%BE20.3.15%20%E5%B0%86%20lib.tar.gz%20%E8%A7%A3%E5%8E%8B%E5%88%B0usrlib%20%E7%9B%AE.png)

> ⚠️ **大坑（原书已提示）**：删掉出厂 `libjpeg` 后，板子重启时出厂 **Qt GUI 里的图片会显示空白**——因为 Qt 解码 jpeg 底层就用 libjpeg，且版本绑定，即使把新库拷进去也救不了 Qt。
> **我的做法（推荐）**：不动 `/usr/lib` 的出厂库，把 libjpeg 装到板子独立目录 `/home/root/jpeg_lib/{lib,bin}`，程序运行时用 `export LD_LIBRARY_PATH=/home/root/jpeg_lib/lib:$LD_LIBRARY_PATH` 指过去。这样既用上新版库，又不破坏 Qt。

验证移植是否成功：

```bash
djpeg --help
```

![20-15 图20.3.16 测试 libjpeg 移植是否成功](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-15%20%E5%9B%BE20.3.16%20%E6%B5%8B%E8%AF%95%20libjpeg%20%E7%A7%BB%E6%A4%8D%E6%98%AF%E5%90%A6%E6%88%90%E5%8A%9F.png)

`djpeg` 是编译源码得到的测试工具（在安装目录 `bin` 下，不是 `lib`）。能打印帮助信息 = 移植成功。

---

## 20.4 libjpeg 使用说明（解码核心）

应用里只需 `#include <jpeglib.h>`。

**解码 8 步：**

1. 创建解码对象
2. 指定数据源
3. 读取图像信息
4. 设置解码参数
5. 开始解码
6. 读取解码后的数据
7. 解码完毕
8. 释放/销毁解码对象

两个核心结构体：

```c
struct jpeg_decompress_struct cinfo;   // 解码对象：记录 jpeg 详细信息 + 输出数据信息
struct jpeg_error_mgr jerr;            // 错误处理对象
```

### 20.4.1 错误处理

libjpeg 有默认错误处理：内存不足、文件格式不对等 → 调用默认错误函数 → **`exit()` 结束整个进程**。

`jpeg_std_error()` 把错误处理设为默认方式，并与解压对象绑定：

```c
cinfo.err = jpeg_std_error(&jerr);
```

`error_exit` 函数指针指向真正的错误处理函数。想自定义可自己写并替换：

```c
void my_error_exit(struct jpeg_decompress_struct *cinfo) { /* ... */ }
cinfo.err.error_exit = my_error_exit;
```

`jpeg_error_mgr` 主要成员（了解即可）：`error_exit`、`emit_message`、`output_message`、`format_message`、`msg_code`、`msg_parm`、`trace_level`、`num_warnings`、`jpeg_message_table`、`last_jpeg_message`、`addon_message_table`、`first_addon_message`、`last_addon_message`。

### 20.4.2 创建解码对象

必做：

```c
jpeg_create_decompress(&cinfo);
```

用完（正常结束或出错）必须销毁，否则内存泄漏：

```c
jpeg_destroy_decompress(&cinfo);
```

### 20.4.3 设置数据源

用标准 I/O 打开文件并指定：

```c
FILE *jpeg_file = NULL;
jpeg_file = fopen("./image.jpg", "r");   // 只读
if (NULL == jpeg_file) {
    perror("fopen error");
    return -1;
}
jpeg_stdio_src(&cinfo, jpeg_file);
```

数据源也可以来自**内存**，不一定是文件流。

### 20.4.4 读取 jpeg 头信息

必做（约定）：

```c
jpeg_read_header(&cinfo, TRUE);
```

读完可拿到：

```c
cinfo.image_width          // jpeg 图像宽度
cinfo.image_height         // jpeg 图像高度
cinfo.num_components       // 颜色通道数
cinfo.jpeg_color_space     // 颜色空间
```

颜色空间 `J_COLOR_SPACE` 枚举：`JCS_UNKNOWN`、`JCS_GRAYSCALE`（灰度）、`JCS_RGB`、`JCS_YCbCr`（YUV）、`JCS_CMYK`、`JCS_YCCK`、`JCS_BG_RGB`、`JCS_BG_YCC`。

### 20.4.5 设置解码参数

参数都有默认值，`jpeg_read_header()` 后已按默认填好，直接改 `cinfo` 成员即可。两个代表参数：

- **输出颜色 `cinfo.out_color_space`**：默认 `JCS_RGB`。
- **缩放 `cinfo.scale_num` / `cinfo.scale_denom`**：输出大小 = `scale_num/scale_denom`，JPEG **仅支持 1/1、1/2、1/4、1/8** 这几种缩小比例，默认 1/1。例：输出原图 1/2：

```c
cinfo.scale_num = 1;
cinfo.scale_denom = 2;
```

> 缩放只支持 1/2 这种整数分之一，**不能任意缩放**。想显示大图，要么先在外面缩图，要么用 1/2、1/4、1/8。

### 20.4.6 开始解码

```c
jpeg_start_decompress(&cinfo);
```

调用后输出参数才被填充：

```c
cinfo.output_width         // 输出宽度
cinfo.output_height        // 输出高度
cinfo.output_components    // 每像素通道数（灰度 1，RGB888 为 3）
```

想**在 `jpeg_start_decompress` 之前**就知道这些值，先调 `jpeg_calc_output_dimensions()`。

### 20.4.7 读取解码后的数据

数据**按行**读取，从左到右、从上到下；每个像素的各通道数据依次存放。例如 24-bit RGB 一行存储为 `B,G,R,B,G,R,...`（原书原话）。

libjpeg 默认输出 **BGR888**（原书说法：R 在低 8 位、B 在高 8 位）。定义 BGR888 类型：

```c
typedef struct bgr888_color {
    unsigned char red;
    unsigned char green;
    unsigned char blue;
} __attribute__ ((packed)) bgr888_t;
```

> `__attribute__((packed))` 必须有，否则结构体会被对齐填充，按字节解释时就错了。

分配行缓冲：大小 = 输出宽度 × 每像素字节数（RGB 是 ×3，灰度 ×1）：

```c
bgr888_t *line_buf = malloc(cinfo.output_width * cinfo.output_components);
```

读一行：

```c
jpeg_read_scanlines(&cinfo, &buf, 1);   // 目前只支持一次 1 行
```

`cinfo.output_scanline` 是"下一行要读的索引"，初始 0，每读一行自增。循环读全部：

```c
while (cinfo.output_scanline < cinfo.output_height) {
    jpeg_read_scanlines(&cinfo, buffer, 1);
    // do something
}
```

### 20.4.8 结束解码

```c
jpeg_finish_decompress(&cinfo);
```

### 20.4.9 销毁对象

```c
jpeg_destroy_decompress(&cinfo);
```

---

## 20.5 libjpeg 应用编程（实战）

例程路径：`20_libjpeg/show_jpeg_image.c`。功能：解码指定 jpg，显示到 LCD。

```c
/***************************************************************
 Copyright © ALIENTEK Co., Ltd. 1998-2021. All rights reserved.
 文件名 : show_jpeg_image.c
 作者 : 邓涛
 版本 : V1.0
 描述 : libjpeg 使用实战
 其他 : 无
 论坛 : www.openedv.com
 日志 : 初版 V1.0 2021/6/15 邓涛创建
 ***************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <string.h>
#include <linux/fb.h>
#include <sys/mman.h>
#include <jpeglib.h>

typedef struct bgr888_color {
    unsigned char red;
    unsigned char green;
    unsigned char blue;
} __attribute__ ((packed)) bgr888_t;

static int width;                          // LCD X 分辨率
static int height;                         // LCD Y 分辨率
static unsigned short *screen_base = NULL; // 映射后的显存基地址
static unsigned long line_length;          // LCD 一行的长度（字节）
static unsigned int bpp;                   // 像素深度 bpp

static int show_jpeg_image(const char *path)
{
    struct jpeg_decompress_struct cinfo;
    struct jpeg_error_mgr jerr;
    FILE *jpeg_file = NULL;
    bgr888_t *jpeg_line_buf = NULL;         // 存从 jpeg 解出的一行
    unsigned short *fb_line_buf = NULL;     // 存要写入 LCD 的一行
    unsigned int min_h, min_w;
    unsigned int valid_bytes;
    int i;

    /* 绑定默认错误处理函数 */
    cinfo.err = jpeg_std_error(&jerr);

    /* 打开 .jpeg/.jpg 图像文件 */
    jpeg_file = fopen(path, "r");
    if (NULL == jpeg_file) {
        perror("fopen error");
        return -1;
    }

    /* 创建 JPEG 解码对象 */
    jpeg_create_decompress(&cinfo);

    /* 指定图像文件 */
    jpeg_stdio_src(&cinfo, jpeg_file);

    /* 读取图像信息 */
    jpeg_read_header(&cinfo, TRUE);

    printf("jpeg 图像大小: %d*%d\n", cinfo.image_width, cinfo.image_height);

    /* 设置解码参数 */
    cinfo.out_color_space = JCS_RGB;        // 默认就是 JCS_RGB
    //cinfo.scale_num = 1;
    //cinfo.scale_denom = 2;

    /* 开始解码图像 */
    jpeg_start_decompress(&cinfo);

    /* 为缓冲区分配内存空间 */
    jpeg_line_buf = malloc(cinfo.output_components * cinfo.output_width);
    fb_line_buf = malloc(line_length);

    /* 判断图像和 LCD 屏哪个的分辨率更低 */
    if (cinfo.output_width > width)
        min_w = width;
    else
        min_w = cinfo.output_width;

    if (cinfo.output_height > height)
        min_h = height;
    else
        min_h = cinfo.output_height;

    /* 读取数据 */
    valid_bytes = min_w * bpp / 8;          // 真正写入显存的一行字节数
    while (cinfo.output_scanline < min_h) {
        jpeg_read_scanlines(&cinfo, (unsigned char **)&jpeg_line_buf, 1); // 读一行

        /* 将读取到的 BGR888 数据转为 RGB565 */
        for (i = 0; i < min_w; i++)
            fb_line_buf[i] = ((jpeg_line_buf[i].red   & 0xF8) << 8) |
                             ((jpeg_line_buf[i].green & 0xFC) << 3) |
                             ((jpeg_line_buf[i].blue  & 0xF8) >> 3);

        memcpy(screen_base, fb_line_buf, valid_bytes);
        screen_base += width;               // 定位到 LCD 下一行显存起点
    }

    /* 解码完成 */
    jpeg_finish_decompress(&cinfo);
    jpeg_destroy_decompress(&cinfo);

    /* 关闭文件、释放内存 */
    fclose(jpeg_file);
    free(fb_line_buf);
    free(jpeg_line_buf);
    return 0;
}

int main(int argc, char *argv[])
{
    struct fb_fix_screeninfo fb_fix;
    struct fb_var_screeninfo fb_var;
    unsigned int screen_size;
    int fd;

    /* 传参校验 */
    if (2 != argc) {
        fprintf(stderr, "usage: %s <jpeg_file>\n", argv[0]);
        exit(-1);
    }

    /* 打开 framebuffer 设备 */
    if (0 > (fd = open("/dev/fb0", O_RDWR))) {
        perror("open error");
        exit(EXIT_FAILURE);
    }

    /* 获取参数信息 */
    ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);

    line_length = fb_fix.line_length;
    bpp = fb_var.bits_per_pixel;
    screen_size = line_length * fb_var.yres;
    width = fb_var.xres;
    height = fb_var.yres;

    /* 将显示缓冲区映射到进程地址空间 */
    screen_base = mmap(NULL, screen_size, PROT_WRITE, MAP_SHARED, fd, 0);
    if (MAP_FAILED == (void *)screen_base) {
        perror("mmap error");
        close(fd);
        exit(EXIT_FAILURE);
    }

    /* 屏幕刷白，然后显示 jpeg */
    memset(screen_base, 0xFF, screen_size);
    show_jpeg_image(argv[1]);

    /* 退出 */
    munmap(screen_base, screen_size);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

### 代码关键点

1. **`min_w / min_h` 是裁剪，不是缩放**：取"图像尺寸"和"屏幕尺寸"的较小值。图比屏大就**只显示左上角**，多出的部分丢弃；图比屏小就只写图那么大。所以原书代码对超大图只能显示一部分，不会自动缩小。
2. **RGB565 转换**：
   ```c
   R5 = (r & 0xF8) << 8;   G6 = (g & 0xFC) << 3;   B5 = (b & 0xF8) >> 3;
   ```
   合起来 16 位：`RRRRR GGGGGG BBBBB`。
3. **`valid_bytes = min_w * bpp / 8`**：16bpp 时就是 `min_w * 2`。
4. **`screen_base += width`**：`screen_base` 是 `unsigned short *`，`+= width` 等于前进 `width × 2` 字节 = 一行。**这默认 LCD 是 16bpp、且 `line_length == width × 2`**（IMX6ULL 出厂 RGB565 屏正是如此，实测 stride=2048、xres=1024），写屏本身是对的。
   ⚠️ **但原书这里有隐藏 bug**：`screen_base` 是**全局变量**，写屏循环把它推到映射区末尾后，`main()` 的 `munmap(screen_base, screen_size)` 会拿到偏移地址 → 段错误。**必须改成局部游标**（见下方"踩坑记录"）。
5. **`__attribute__((packed))`**：保证 `bgr888_t` 三字节紧凑排列。

> 命名说明：结构体叫 `bgr888_t`、字段却是 `red/green/blue`，是历史叫法。libjpeg 在 `out_color_space = JCS_RGB` 时按 R、G、B 顺序输出，结构体按字节顺序正好对上，所以 `.red` 取到的是 R。别被名字绕晕。

### 编译

原书写法（`${CC}` 为环境脚本导出的交叉编译器变量）：

```bash
${CC} -o testApp testApp.c -I /home/dt/tools/jpeg/include -L /home/dt/tools/jpeg/lib -ljpeg
```

![20-16 图20.5.1 编译示例代码](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-16%20%E5%9B%BE20.5.1%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81%20%E7%BC%96%E8%AF%91%E7%9A%84%E6%97%B6%E5%80%99%E9%9C%80%E8%A6%81%E6%8C%87%E5%AE%9A%E5%A4%B4%E6%96%87%E4%BB%B6%E7%9A%84%E8%B7%AF%E5%BE%84%E5%BA%93%E6%96%87.png)

用法一致：`-I` 头文件目录、`-L` 库目录、`-ljpeg` 链接 libjpeg。

**用独立工具链时**（实测可用）：

```bash
arm-linux-gnueabihf-gcc -o show_jpeg_image show_jpeg_image.c \
    -I ~/tools/jpeg/include -L ~/tools/jpeg/lib -ljpeg
```

### 运行

把可执行文件 + 一张 `.jpg` 拷到板子家目录（`/home/root`，**scp 时务必带文件名**，否则会把子目录替换成文件）：

```bash
scp show_jpeg_image image.jpg root@<板子IP>:/home/root/
```

板子上执行（用独立库目录时先设环境变量）：

```bash
export LD_LIBRARY_PATH=/home/root/jpeg_lib/lib:$LD_LIBRARY_PATH
./show_jpeg_image image.jpg
```

![20-17 图20.5.2 执行应用程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-17%20%E5%9B%BE20.5.2%20%E6%89%A7%E8%A1%8C%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%20%E6%AD%A4%E6%97%B6%20LCD%20%E5%B1%8F%E4%B8%8A%E4%BE%BF%E4%BC%9A%E6%98%BE%E7%A4%BA.png)

执行前建议**关闭出厂 Qt GUI**，否则会抢显存/刷新。

![20-18 图20.5.3 LCD 显示效果](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BAjpeg%E5%9B%BE%E5%83%8F/20-18%20%E5%9B%BE20.5.3%20LCD%20%E6%98%BE%E7%A4%BA%E6%95%88%E6%9E%9C.jpg)

### 实测踩坑记录

- **段错误（rc=139）真因（已实测确认）**：**不是图太大，也不是 bpp 问题**。`screen_base` 是全局变量，`show_jpeg_image()` 每写一行 `screen_base += width` 把它推到映射区末尾；回到 `main()` 再 `munmap(screen_base, screen_size)` 用的是**偏移后的地址**，映射没被正确解除，收尾时踩空 → SIGSEGV。`strace` 里能看到 `munmap(...)` 成功后紧跟 `SIGSEGV SEGV_MAPERR`。
  **修复**：写屏用局部游标，全局 `screen_base` 只保留 mmap 原始基址给 `munmap` 用：
  ```c
  unsigned short *fb_ptr = screen_base;   // 函数内新增
  ...
  memcpy(fb_ptr, fb_line_buf, valid_bytes);
  fb_ptr += width;                        // 原来的 screen_base += width 改这里
  ```
  改完重编上板：`rc=0`，1024×600 正常显示（板子 16bpp、stride=2048）。
- **`too few scanlines`**：喂明显大于屏幕的图（如 4096×2731）且只读 `min_h` 行就 `jpeg_finish_decompress` 时会报，属原书裁剪逻辑局限。**解决**：先把图缩到屏幕内，或用 `scale_num/scale_denom=1/8`。
- **Qt 图片空白**：删 `/usr/lib/libjpeg.*` 的副作用，见 20.3.4，用独立目录避坑。
- **`cjpeg`/`djpeg` 是 ARM 程序**，在 Ubuntu 上不能直接跑，要在板子上跑。

---

## 20.6 总结

- libjpeg 不止解码，还能编码，API 不多，打开 `jpeglib.h` 浏览函数名 + 注释就能大致知道用途。
- 学习方法是"以点带面"：掌握本章这套 8 步解码流程，其他库（alsa-lib 等）同理。
- 解码流程务必记住顺序，尤其 `jpeg_create_decompress` / `jpeg_destroy_decompress`、`jpeg_start_decompress` / `jpeg_finish_decompress` 要成对出现。
- 下一章：用 **libpng + zlib** 在 LCD 上显示 png 图片。
