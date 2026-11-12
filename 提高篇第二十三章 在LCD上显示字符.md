# 提高篇第二十三章 在 LCD 上显示字符

> 配套例程：`23_freetype/show_char.c`（取模）、`23_freetype/freetype_test.c`（FreeType）
> 两种思路：**① 自己取模打点**（原理清楚，但只适合固定几个字）；**② 用 FreeType 解析字体文件**（工程做法）。

本章主题：
- 原始方式：自己取模显示字符
- 使用 FreeType 访问字体文件（简介、移植、使用）

---

## 23.1 原始方式：取模显示字符

### 原理

LCD 由 `width × height` 个像素点组成。显示字符最朴素的办法：**把字符做成点阵，再逐点打到屏幕**。

![23-1 图23.1.1 字符点阵](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-1%20%E5%9B%BE23.1.1%20%E5%AD%97%E7%AC%A6%E7%82%B9%E9%98%B5.png)

以"正"字为例，点阵 **宽 64、高 86**（64 个小方块 × 86 行）。用一个二维数组存：

```c
unsigned char arr[86][8];   // 86 行，每行 64 bit = 8 字节
```

- 一行 64 个方块 = 8 个字节（8×8=64 bit）；
- **填充颜色的方块记 1，不填充记 0**，一个方块对应一个 bit。

取模软件选择"**从左到右、从上到下**"，代码就按这个顺序解析。

### 程序

源码：`23_freetype/show_char.c`。4 个数组 `ch_char1..4` 是"正点原子"4 个字的 64×86 子模（每个 86×8 字节）。

```c
#define argb8888_to_rgb565(color) ({ \
    unsigned int temp = (color); \
    ((temp & 0xF80000UL) >> 8) | \
    ((temp & 0xFC00UL) >> 5) | \
    ((temp & 0xF8UL) >> 3); \
})

static void lcd_draw_character(unsigned int x, unsigned int y,
                               const unsigned char *ch,
                               unsigned int w, unsigned int h,
                               unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    unsigned long temp;
    unsigned int end_x, end_y;
    int j;
    int columns;

    /* 宽度不是 8 的整数倍要向上取整 */
    columns = w / 8;
    if (0 != w % 8)
        columns++;

    if (w < 1 || h < 1)
        return;
    if (x >= width || y >= height)
        return;

    end_x = x + w - 1;
    end_y = y + h - 1;
    if (end_x >= width)  end_x = width - 1;
    if (end_y >= height) end_y = height - 1;

    h = end_y - y + 1;
    w = end_x - x + 1;

    temp = y * width + x;                 // 定位到字符左上角
    for (y = 0; y < h; y++, temp += width) {
        for (x = 0, j = 0; x < w; ) {
            if (*(ch + y * columns + j) & (0x1 << (x % 8)))
                screen_base[temp + x] = rgb565_color;
            x++;
            if (0 == x % 8) j++;          // 每 8 个点换一个字节
        }
    }
}
```

`main()` 里刷白背景，然后把 4 个字居中排开：

```c
memset(screen_base, 0xFF, screen_size);

int x = width * 0.5 - 128;   // 4 个字总宽 256
int y = height * 0.5 - 43;   // 字高 86，居中

lcd_draw_character(x,       y, (unsigned char *)ch_char1, 64, 86, 0xFF00FF);
lcd_draw_character(x + 64,  y, (unsigned char *)ch_char2, 64, 86, 0xFF00FF);
lcd_draw_character(x + 128, y, (unsigned char *)ch_char3, 64, 86, 0xFF00FF);
lcd_draw_character(x + 192, y, (unsigned char *)ch_char4, 64, 86, 0xFF00FF);
```

> 字模数组很长（每个 86 行），笔记里不展开，完整数据见源码 `show_char.c`。

编译（不需要额外库）：

```bash
${CC} -o show_char show_char.c
```

![23-2 图23.1.2 执行测试程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-2%20%E5%9B%BE23.1.2%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%20%E6%AD%A4%E6%97%B6%E5%BC%80%E5%8F%91%E6%9D%BF%20LCD%20%E5%B1%8F%E4%B8%8A%E4%BC%9A%E6%98%BE%E7%A4%BA%E2%80%9C%E6%AD%A3.png)

![23-3 图23.1.3 LCD 屏显示中文字符](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-3%20%E5%9B%BE23.1.3%20LCD%20%E5%B1%8F%E6%98%BE%E7%A4%BA%E4%B8%AD%E6%96%87%E5%AD%97%E7%AC%A6.jpg)

这种方式简单，但很"low"：只能显示取好模的那几个字，实际项目不会这么干（除非就显示固定几个字符）。

### 为什么还要讲

1. 理解字符显示的本质：**点阵 + 打点**；
2. 有些项目确实只用固定字符，取模反而最省资源。

### 字体文件

操作系统里通常有大量字体文件：
- Windows：`C:\Windows\Fonts`（otf、ttf、ttc 等）
- Linux：`/usr/share/fonts`

![23-4 图23.1.4 Windows 系统中的字体文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-4%20%E5%9B%BE23.1.4%20Windows%20%E7%B3%BB%E7%BB%9F%E4%B8%AD%E7%9A%84%E5%AD%97%E4%BD%93%E6%96%87%E4%BB%B6.png)

有了字体文件，字符的位图已经编码在里面，**不用再取模**，只需解析字体文件。解析工作交给 FreeType。

---

## 23.2 FreeType 简介

- **FreeType** 是完全免费开源的软件字体引擎，设计小巧、高效、可移植。
- 提供**统一接口**访问多种字体格式（ttf、ttc、otf 等）。
- 注意叫 **FreeType 2**（区别于已废弃的 FreeType 1）。

---

## 23.3 FreeType 移植

### 下载源码

出厂系统有 FreeType 2.6，版本偏低，书里移植 **2.8**。

地址：<https://download.savannah.gnu.org/releases/freetype/>

![23-5 图23.3.1 FreeType 源码下载地址](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-5%20%E5%9B%BE23.3.1%20FreeType%20%E6%BA%90%E7%A0%81%E4%B8%8B%E8%BD%BD%E5%9C%B0%E5%9D%80%20%E5%BE%80%E4%B8%8B%E7%BF%BB%E6%89%BE%E5%88%B0%202..png)

选 `freetype-2.8.tar.gz`：

![23-6 图23.3.2 找到 2.8 版本](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-6%20%E5%9B%BE23.3.2%20%E6%89%BE%E5%88%B0%202.8%20%E7%89%88%E6%9C%AC%20%E7%82%B9%E5%87%BB%20freetype-2..png)

### 编译

拷到 Ubuntu 家目录，建安装目录：

```bash
mkdir -p ~/tools/freetype
tar -xzf freetype-2.8.tar.gz
cd freetype-2.8
```

![23-7 图23.3.4 将源码包拷贝到 Ubuntu](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-7%20%E5%9B%BE23.3.4%20%E5%B0%86%E6%BA%90%E7%A0%81%E5%8C%85%E6%8B%B7%E8%B4%9D%E5%88%B0%20Ubuntu%20%E7%B3%BB%E7%BB%9F%20%E5%9C%A8%20tool.png)

![23-8 图23.3.5 创建 FreeType 安装目录](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-8%20%E5%9B%BE23.3.5%20%E5%88%9B%E5%BB%BA%20FreeType%20%E7%9A%84%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%20%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4%E5%B0%86.png)

![23-9 图23.3.6 解压](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-9%20%E5%9B%BE23.3.6%20%E5%B0%86%20freetype-2.8.tar.gz%20%E5%8E%8B%E7%BC%A9.png)

**裁剪/配置**：FreeType 模块化，可以裁。书里只打开两个宏（`include/freetype/config/ftoption.h`）：

```c
#define FT_CONFIG_OPTION_SYSTEM_ZLIB   // 使用系统 zlib（支持 Gzip）
#define FT_CONFIG_OPTION_USE_PNG       // 支持 PNG 位图字形（依赖 libpng）
```

![23-10 图23.3.7 ftoption.h 配置文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-10%20%E5%9B%BE23.3.7%20ftoption.h%20%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6.png)

配置命令（重点）：

```bash
. /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi

./configure --prefix=$HOME/tools/freetype \
  --host=arm-poky-linux-gnueabi \
  --with-zlib=yes --with-bzip2=no --with-png=yes --with-harfbuzz=no \
  ZLIB_CFLAGS="-I$HOME/tools/zlib/include -L$HOME/tools/zlib/lib" ZLIB_LIBS=-lz \
  LIBPNG_CFLAGS="-I$HOME/tools/png/include -L$HOME/tools/png/lib" LIBPNG_LIBS=-lpng
```

参数含义：

| 参数 | 作用 |
|------|------|
| `--prefix` | 安装目录 |
| `--host` | 交叉编译器前缀 |
| `--with-zlib=yes` | 用 zlib |
| `--with-bzip2=no` | 不用 bzip2 |
| `--with-png=yes` | 用 libpng |
| `--with-harfbuzz=no` | 不用 harfbuzz |
| `ZLIB_CFLAGS/LIBS` | zlib 头文件/库路径、链接名 |
| `LIBPNG_CFLAGS/LIBS` | libpng 头文件/库路径、链接名 |

![23-11 图23.3.8 配置 FreeType 工程](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-11%20%E5%9B%BE23.3.8%20%E9%85%8D%E7%BD%AE%20FreeType%20%E5%B7%A5%E7%A8%8B%20%E9%85%8D%E7%BD%AE%E5%AE%8C%E6%88%90%E4%B9%8B%E5%90%8E%E6%8E%A5%E7%9D%80%E6%89%A7.png)

```bash
make -j4
make install
```

### 安装目录与使用头文件

安装目录同样有 `bin`、`include`、`lib`：

![23-12 图23.3.9 FreeType 安装目录下的文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-12%20%E5%9B%BE23.3.9%20FreeType%20%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%20%E5%90%8C%E6%A0%B7%E6%9C%89%20bi.png)

应用里这样包含头文件：

```c
#include <ft2build.h>
#include FT_FREETYPE_H   // 实际是 freetype2/freetype/freetype.h
```

### 部署到开发板

拷 `lib` 下库文件到板子 `/usr/lib`，先删出厂：

```bash
rm -rf /usr/lib/libfreetype.*
```

再拷新库（注意符号链接）。

![23-13 图23.3.10 开发板 /usr/lib 目录下的 FreeType](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-13%20%E5%9B%BE23.3.10%20%E5%BC%80%E5%8F%91%E6%9D%BFusrlib%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%20FreeType.png)

> ⚠️ 和第 20 章一样：删系统库会影响出厂 Qt。**推荐独立目录**（如 `/home/root/freetype_lib`）+ `LD_LIBRARY_PATH`。

---

## 23.4 FreeType 库的使用

官方文档：
- <https://www.freetype.org/freetype2/docs/tutorial/step1.html>
- <https://www.freetype.org/freetype2/docs/tutorial/step2.html>
- <https://www.freetype.org/freetype2/docs/reference/index.html>

### 核心概念

| 概念 | 含义 |
|------|------|
| 字形 glyph | 字符的图像；同一字符不同字体是不同字形 |
| 字形索引 | 字体文件里用索引找字形，索引由字符编码（ASCII/GBK/Unicode）转换而来 |
| 点 point | 物理单位，1 point = 1/72 英寸 |
| dpi | 每英寸像素数 |
| 像素数 | `像素数 = 点数 × dpi / 72`（例：50pt @300dpi = 50×300/72 ≈ 208px） |
| 布局 | 水平布局（横写）/ 垂直布局（竖写，如对联） |
| 原点/基线 | 水平布局用水平基线（上下对齐），垂直基线（左右对齐） |
| width / height | 字形轮廓的宽高 |
| bearingX | 垂直基线到字形轮廓最左边的距离 |
| bearingY | 水平基线到字形轮廓最上边的距离 |
| xMin/xMax/yMin/yMax | 字形边界框（bbox）四边位置 |
| advance | 步进宽度，相邻两个原点的距离（字间距） |

![23-14 图23.4.1 水平布局](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-14%20%E5%9B%BE23.4.1%20%E6%B0%B4%E5%B9%B3%E5%B8%83%E5%B1%80.png)

![23-15 图23.4.2 垂直布局](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-15%20%E5%9B%BE23.4.2%20%E5%9E%82%E7%9B%B4%E5%B8%83%E5%B1%80.png)

### 位图数据

FreeType 取到的字形位图放在 buffer 里，大小 = **字形宽 × 高** 字节（每点 1 字节）：

- 值 = 0：不填充
- 值 > 0：填充（值越大越"实"，可以拿来做抗锯齿/灰度）

### 怎么对齐

靠**基线**对齐：
- 水平基线管上下对齐（不同字符 bearingY 不同，靠它统一）；
- 垂直基线管左右对齐；
- 相邻垂直基线距离 = advance。

**左上角定位公式**（关键）：

```
左上角 = (原点x + bearingX, 原点y − bearingY)
```

![23-16 图23.4.3 通过水平基线和垂直基线保证对齐显示](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-16%20%E5%9B%BE23.4.3%20%E9%80%9A%E8%BF%87%E6%B0%B4%E5%B9%B3%E5%9F%BA%E7%BA%BF%E5%92%8C%E5%9E%82%E7%9B%B4%E5%9F%BA%E7%BA%BF%E4%BF%9D%E8%AF%81%E5%AF%B9%E9%BD%90%E6%98%BE%E7%A4%BA%20%E4%B8%8D%E5%90%8C%E5%AD%97%E7%AC%A6%E5%AF%B9%E5%BA%94.png)

### 初始化

```c
FT_Library library;
FT_Error error;

error = FT_Init_FreeType(&library);
if (error)
    fprintf(stderr, "failed to init FreeType\n");
```

成功返回 0，失败返回非 0 错误码。

### 加载 face（字体文件）

```c
FT_Face face;
error = FT_New_Face(library, "/usr/share/fonts/font.ttf", 0, &face);
```

```c
FT_Error FT_New_Face(FT_Library library, const char *filepathname,
                     FT_Long face_index, FT_Face *aface);
```

| 参数 | 说明 |
|------|------|
| `library` | 库句柄 |
| `filepathname` | 字体文件路径 |
| `face_index` | 一个文件可含多个 face，通常填 0；`face->num_faces` 可查总数 |
| `aface` | 返回的 face 指针，失败为 NULL |

### 设置字体大小

**方式一：按像素**

```c
FT_Set_Pixel_Sizes(face, 50, 50);
FT_Set_Pixel_Sizes(face, 50, 0);   // 高=0 表示与宽相等
```

**方式二：按点（1/64 point，26.6 定点）**

```c
FT_Set_Char_Size(face,
                 16 * 64,   // 宽，1/64 点
                 16 * 64,   // 高
                 300,       // 水平 dpi
                 300);      // 垂直 dpi
```

- 宽/高任一设为 0，则与另一个相等；
- dpi 设 0 → 默认 72。

### 加载字形

```c
FT_Get_Char_Index(face, ch);              // 编码 -> 字形索引
FT_Load_Glyph(face, index, FT_LOAD_RENDER); // 按索引加载
FT_Load_Char(face, ch, FT_LOAD_RENDER);     // 编码直接加载（最常用）
```

`FT_LOAD_RENDER` 会直接把字形渲染成位图，结果在 `face->glyph`（`FT_GlyphSlot`）：

```c
slot = face->glyph;
slot->bitmap.buffer;   // 位图数据
slot->bitmap.width;    // 位图宽
slot->bitmap.rows;     // 位图高（行数）
slot->bitmap_left;     // bearingX
slot->bitmap_top;      // bearingY
slot->advance.x;       // 步进（26.6 定点，除以 64 得像素）
```

---

## 23.5 示例代码

源码：`23_freetype/freetype_test.c`。功能：加载字体文件，在 LCD 上显示 4 行中文诗，支持旋转/斜体角度参数。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <string.h>
#include <errno.h>
#include <sys/mman.h>
#include <linux/fb.h>
#include <math.h>
#include <wchar.h>
#include <ft2build.h>
#include FT_FREETYPE_H

#define FB_DEV "/dev/fb0"

#define argb8888_to_rgb565(color) ({ \
    unsigned int temp = (color); \
    ((temp & 0xF80000UL) >> 8) | \
    ((temp & 0xFC00UL) >> 5) | \
    ((temp & 0xF8UL) >> 3); \
})

static unsigned int width;
static unsigned int height;
static unsigned short *screen_base = NULL;
static unsigned long screen_size;
static int fd = -1;

static FT_Library library;
static FT_Face face;

static int fb_dev_init(void)
{
    struct fb_var_screeninfo fb_var = {0};
    struct fb_fix_screeninfo fb_fix = {0};

    fd = open(FB_DEV, O_RDWR);
    if (0 > fd) {
        fprintf(stderr, "open error: %s: %s\n", FB_DEV, strerror(errno));
        return -1;
    }

    ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);

    screen_size = fb_fix.line_length * fb_var.yres;
    width = fb_var.xres;
    height = fb_var.yres;

    screen_base = mmap(NULL, screen_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (MAP_FAILED == (void *)screen_base) {
        perror("mmap error");
        close(fd);
        return -1;
    }

    memset(screen_base, 0xFF, screen_size);
    return 0;
}

static int freetype_init(const char *font, int angle)
{
    FT_Error error;
    FT_Vector pen;
    FT_Matrix matrix;
    float rad;

    FT_Init_FreeType(&library);

    error = FT_New_Face(library, font, 0, &face);
    if (error) {
        fprintf(stderr, "FT_New_Face error: %d\n", error);
        exit(EXIT_FAILURE);
    }

    pen.x = 0 * 64;
    pen.y = 0 * 64;

    rad = (1.0 * angle / 180) * M_PI;

#if 1
    /* 斜体 */
    matrix.xx = (FT_Fixed)( cos(rad) * 0x10000L);
    matrix.xy = (FT_Fixed)( sin(rad) * 0x10000L);
    matrix.yx = (FT_Fixed)( 0 * 0x10000L);
    matrix.yy = (FT_Fixed)( 1 * 0x10000L);
#else
    /* 旋转 */
    matrix.xx = (FT_Fixed)( cos(rad) * 0x10000L);
    matrix.xy = (FT_Fixed)(-sin(rad) * 0x10000L);
    matrix.yx = (FT_Fixed)( sin(rad) * 0x10000L);
    matrix.yy = (FT_Fixed)( cos(rad) * 0x10000L);
#endif

    FT_Set_Transform(face, &matrix, &pen);
    FT_Set_Pixel_Sizes(face, 40, 0);

    return 0;
}

static void lcd_draw_character(int x, int y, const wchar_t *str, unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    FT_GlyphSlot slot = face->glyph;
    size_t len = wcslen(str);
    long int temp;
    int n;
    int i, j, p, q;
    int max_x, max_y, start_y, start_x;

    for (n = 0; n < len; n++) {

        if (FT_Load_Char(face, str[n], FT_LOAD_RENDER))
            continue;

        /* 左上角 y = 原点 - bearingY */
        start_y = y - slot->bitmap_top;
        if (0 > start_y) {          // 超出上边界
            q = -start_y;
            temp = 0;
            j = 0;
        } else {
            q = 0;
            temp = width * start_y;
            j = start_y;
        }

        max_y = start_y + slot->bitmap.rows;
        if (max_y > (int)height)
            max_y = height;

        for (; j < max_y; j++, q++, temp += width) {

            /* 左上角 x = 原点 + bearingX */
            start_x = x + slot->bitmap_left;
            if (0 > start_x) {
                p = -start_x;
                i = 0;
            } else {
                p = 0;
                i = start_x;
            }

            max_x = start_x + slot->bitmap.width;
            if (max_x > (int)width)
                max_x = width;

            for (; i < max_x; i++, p++) {
                if (slot->bitmap.buffer[q * slot->bitmap.width + p])
                    screen_base[temp + i] = rgb565_color;
            }
        }

        /* 下一个字：水平步进 */
        x += slot->advance.x / 64;
        y -= slot->advance.y / 64;
    }
}

int main(int argc, char *argv[])
{
    if (3 != argc) {
        fprintf(stderr, "usage: %s <font_file> <angle>\n", argv[0]);
        return -1;
    }

    if (fb_dev_init())
        exit(EXIT_FAILURE);

    if (freetype_init(argv[1], atoi(argv[2])))
        exit(EXIT_FAILURE);

    int y = height * 0.25;
    lcd_draw_character(30, 80, L"路漫漫其修远兮，吾将上下而求索", 0x000000);
    lcd_draw_character(30, y + 80, L"莫愁前路无知己，天下谁人不识君", 0x9900FF);
    lcd_draw_character(30, 2 * y + 80, L"君不见黄河之水天上来，奔流到海不复回", 0xFF0099);
    lcd_draw_character(30, 3 * y + 80, L"君不见高堂明镜悲白发，朝如青丝暮成雪", 0x9932CC);

    FT_Done_Face(face);
    FT_Done_FreeType(library);
    munmap(screen_base, screen_size);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

### 代码关键点

1. **`FT_LOAD_RENDER`**：加载并渲染，直接拿 `slot->bitmap`。
2. **左上角定位**：`start_y = y - slot->bitmap_top`、`start_x = x + slot->bitmap_left`，这是全书最容易写错的地方。
3. **越界处理**：`start_y < 0` 时用 `q` 跳过位图顶部；`start_x < 0` 时用 `p` 跳过左侧。
4. **换字用 advance**：`x += slot->advance.x / 64`（26.6 定点转像素）。
5. **中文用 `wchar_t` + `L"..."`**：FreeType 默认 Unicode，字符串要用宽字符。
6. **`FT_Set_Transform`** 可做斜体/旋转，角度参数就是 `argv[2]`。

### 编译

```bash
${CC} -o freetype_test freetype_test.c \
    -I $HOME/tools/freetype/include/freetype2 \
    -L $HOME/tools/freetype/lib -lfreetype \
    -L $HOME/tools/zlib/lib -lz \
    -L $HOME/tools/png/lib -lpng -lm
```

要点：`-I` 到 `freetype2` 目录；要链接 **freetype + z + png + m** 四个库。

![23-18 图23.5.1 编译应用程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-18%20%E5%9B%BE23.5.1%20%E7%BC%96%E8%AF%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%20%E7%A7%BB%E6%A4%8D%E7%9A%84%20FreeType%20%E4%B8%8E%E5%BC%80%E5%8F%91%E6%9D%BF.jpg)

### 字体文件

移植的 FreeType 与板子自带字体可能不匹配，从 Windows `C:\Windows\Fonts` 拷一个字体（书里用宋体 SIMSUN.TTC）。

![23-19 图23.5.2 选择字体](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-19%20%E5%9B%BE23.5.2%20%E9%80%89%E6%8B%A9%E5%AD%97%E4%BD%93%20%E5%B0%86%E7%BC%96%E8%AF%91%E5%A5%BD%E7%9A%84%E7%A8%8B%E5%BA%8F%E4%B8%8E%E5%AD%97%E4%BD%93%E6%96%87%E4%BB%B6%E6%8B%B7%E8%B4%9D%E5%88%B0%E5%BC%80%E5%8F%91%E6%9D%BF.png)

### 运行

```bash
./freetype_test SIMSUN.TTC 0    # 第一个参数字体，第二个参数角度（0=不旋转）
```

![23-20 图23.5.3 执行测试程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-20%20%E5%9B%BE23.5.3%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%20%E7%AC%AC%E4%B8%80%E4%B8%AA%E5%8F%82%E6%95%B0%20SIMSUN.TY%20%E6%98%AF.png)

把角度改成别的值可以看到旋转/斜体效果：

![23-21 p695 显示效果](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-21%20p695%E6%88%AA%E5%9B%BE.jpg)

![23-23 p696 显示效果](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%89%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BA%E5%AD%97%E7%AC%A6/23-23%20p696%E6%88%AA%E5%9B%BE.jpg)

---

## 实测记录（ATK-IMX6U 实机）

### A. 取模方式

- 从原文抽取 4 个 64×86 字模（共 344 行 × 8 字节），生成 `show_char.c`。
- 编译 `rc=0`（15760 字节 ARM），运行 **rc=0**。
- 显存统计：紫红 `0xF81F` **7189** 像素，全部落在居中 `256×86` 区域内（`x=384..640, y=257..343`），背景白 `0xFFFF` 607211 → 字形位置和点阵解析都正确。

### B. FreeType 方式

- 交叉编译 FreeType 2.8：`libfreetype.so.6.14.0`（2.5M），configure 确认 `external zlib: yes`、`libpng: yes`。
- 板子部署：`/home/root/freetype_lib`、字体 `/home/root/uming.ttc`（Ubuntu 自带 AR PL UMing，21M）。
- 编译 `rc=0`（14500 字节 ARM），运行 **rc=0**。
- 显存统计（4 行诗，4 种颜色）：

| 行 | 颜色 | RGB565 | 像素数 |
|----|------|--------|--------|
| 1 | 黑 `0x000000` | `0x0000` | 6399 |
| 2 | 紫 `0x9900FF` | `0x981F` | 5723 |
| 3 | 粉 `0xFF0099` | `0xF813` | 7192 |
| 4 | 紫罗兰 `0x9932CC` | `0x9999` | 8042 |

- 每行非白像素数与其颜色计数完全一致，且分别落在 `y=80/230/380/530`，说明**取字形、bearing 对齐、advance 换字**全部正常。
- 运行命令：
  ```bash
  export LD_LIBRARY_PATH=/home/root/freetype_lib:/home/root/png_lib:/home/root/zlib_lib
  ./freetype_test /home/root/uming.ttc 0
  ```

### 结论

| 方式 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 取模 | 简单、资源少、无需库 | 只能显示固定字符，换字要重新取模 | 固定少数符号 |
| FreeType | 任意字符、任意大小/字体/角度 | 需要移植库 + 字体文件，占资源 | 实际项目 |
