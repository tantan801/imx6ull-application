# 第十九章 FrameBuffer 应用编程

> Framebuffer（帧缓冲）是 Linux 对显示设备的抽象：把 LCD 抽象成一块**显存**，应用层直接读写这块内存就等于在屏幕上画图，不用管硬件怎么扫屏。设备节点是 `/dev/fbX`（`/dev/fb0` 就是 LCD），支持最多 32 个。
>
> 本章是"零移植"的一章——**不需要任何第三方库**，只用 `open` + `ioctl` + `mmap` 就能点亮、画图、显示 BMP。真正需要移植的是后面的 libjpeg/libpng/freetype。
>
> 前提：LCD 排线接好（**断电接线**），出厂系统已把 `/dev/fb0` 配好。

## 19.1 什么是 FrameBuffer

- Frame = 帧，buffer = 缓冲 → **帧缓冲就是保存"一帧图像"的一块内存**。
- Linux 用它作为显示驱动接口，屏蔽不同显示硬件的差异，对应用层抽象成"显存"。
- 应用读写 `/dev/fbX` ≈ 读写显示缓冲区；具体物理地址、扫屏时序都由驱动负责。
- Linux 最多支持 32 个 fb 设备：`/dev/fb0` ~ `/dev/fb31`；开发板出厂系统的 `/dev/fb0` 就是 LCD。

**显存大小怎么算**：`xres × yres × bpp / 8`。例如 800×480、RGB888 → 800×480×24/8 = **1,152,000** 字节。

所以可以用 `dd` 直接清屏（800×480 RGB888 为例）：

```bash
dd if=/dev/zero of=/dev/fb0 bs=1024 count=1125    # 1125×1024 字节全写 0 → 黑屏
```

> 这条命令是**判断 fb 链路是否通**的最快方法：屏幕变黑/变色就说明"应用→fb→LCD"全通。

## 19.2 LCD 的基础知识

本书不展开（时序、RGB 接口、背光等见驱动教程）。应用层只需要记住：**屏幕参数（分辨率、bpp、像素格式）一律通过 ioctl 问驱动要，不要写死**。

## 19.3 LCD 应用编程介绍

操作 `/dev/fbX` 的标准五步：

1. `open("/dev/fbX")` 打开设备；
2. `ioctl` 取参数（分辨率、bpp、像素格式），据此算出显存大小；
3. `mmap` 把显存映射到进程地址空间；
4. 直接读写映射后的内存 → 打点、画线、显示图片；
5. `munmap` 取消映射 + `close` 关闭。

### 用 ioctl 取屏幕参数

三个常用 request（定义在 `<linux/fb.h>`）：

| 宏 | 值 | 作用 | 第三个参数 |
|---|---|---|---|
| `FBIOGET_VSCREENINFO` | 0x4600 | 取**可变**参数 | `struct fb_var_screeninfo *` |
| `FBIOPUT_VSCREENINFO` | 0x4601 | 设**可变**参数 | `struct fb_var_screeninfo *` |
| `FBIOGET_FSCREENINFO` | 0x4602 | 取**固定**参数 | `struct fb_fix_screeninfo *` |

```c
struct fb_var_screeninfo fb_var;
struct fb_fix_screeninfo fb_fix;

ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);   // 分辨率、bpp、RGB 位域
ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);   // line_length、显存地址
```

**`struct fb_var_screeninfo` 关键成员**：

| 成员 | 含义 |
|---|---|
| `xres` / `yres` | 可视区域一行/一列的像素数，即 X/Y 分辨率 |
| `xres_virtual` / `yres_virtual` | 虚拟区域分辨率（可能大于可视区域） |
| `xoffset` / `yoffset` | 虚拟到可见屏幕的行/列偏移 |
| `bits_per_pixel` | 像素深度 bpp：每像素多少 bit |
| `grayscale` | 0 彩色，1 灰度，>1 FOURCC 颜色 |
| `red` / `green` / `blue` / `transp` | 各颜色分量的**位域**（offset + length） |
| `nonstd` | 0 标准像素格式，非 0 非标准 |
| `height` / `width` | 屏幕物理尺寸（mm） |
| `pixclock`、`left/right/upper/lower_margin`、`hsync_len`、`vsync_len` | 时序参数（应用层一般不动） |

**`struct fb_bitfield`**：

```c
struct fb_bitfield {
    __u32 offset;      // 该分量在像素值中的起始位
    __u32 length;      // 该分量占几位
    __u32 msb_right;   // 非 0 表示最高位在右
};
```

靠 `red/green/blue` 的 offset+length 就能判断像素格式：RGB565 是 `R<11,5> G<5,6> B<0,5>`；RGB888 是 `R<16,8> G<8,8> B<0,8>`；BGR 只是偏移顺序不同。

**`struct fb_fix_screeninfo` 关键成员**：

| 成员 | 含义 |
|---|---|
| `id[16]` | 标识字符串 |
| `smem_start` | 显存起始**物理**地址（应用层用不了） |
| `smem_len` | 显存长度，不一定等于实际屏幕显存 |
| `line_length` | **一行的字节数**（关键！算映射长度用它） |
| `type` / `visual` / `xpanstep` / `ypanstep` / `ywrapstep` | 类型、视觉类型、平移步长 |
| `mmio_start` / `mmio_len` | MMIO 物理地址与长度 |

### 例程：lcd_info.c（只读参数）

光盘路径：`11、Linux C应用编程例程源码 -> 19_lcd -> lcd_info.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/fb.h>

int main(int argc, char *argv[])
{
    struct fb_fix_screeninfo fb_fix;
    struct fb_var_screeninfo fb_var;
    int fd;

    /* 打开 framebuffer 设备 */
    if (0 > (fd = open("/dev/fb0", O_WRONLY))) {
        perror("open error");
        exit(-1);
    }

    /* 获取参数信息 */
    ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);

    printf("分辨率: %d*%d\n"
           "像素深度 bpp: %d\n"
           "一行的字节数: %d\n"
           "像素格式: R<%d %d> G<%d %d> B<%d %d>\n",
           fb_var.xres, fb_var.yres,
           fb_var.bits_per_pixel,
           fb_fix.line_length,
           fb_var.red.offset,  fb_var.red.length,
           fb_var.green.offset, fb_var.green.length,
           fb_var.blue.offset,  fb_var.blue.length);

    close(fd);
    exit(0);
}
```

原书 7 寸 800×480 屏的实测输出（图 19.3.1）：

![图19.3.1 获取到屏幕参数](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-1%20%E5%9B%BE19.3.1%20%E8%8E%B7%E5%8F%96%E5%88%B0%E5%B1%8F%E5%B9%95%E5%8F%82%E6%95%B0.png)

解读：

- `800*480` → 分辨率与屏一致；
- `bpp: 16` → 每像素 2 字节；
- `一行的字节数: 1600` → 800 × 16/8 = 1600，吻合；
- `R<11 5> G<5 6> B<0 5>` → 高 5 位 R、中 6 位 G、低 5 位 B，即 **RGB565**。

> **Tips**：正点原子的 RGB 屏硬件都支持 RGB888，但**出厂系统驱动把它实现成 RGB565**。想改 888 得改设备树（驱动层），或在驱动支持的前提下用 `FBIOPUT_VSCREENINFO`——书上明确不建议随便改，驱动不完善会花屏。

### 用 mmap 把显存映射到用户空间

为什么不直接 `read/write`？数据量太大。1920×1080 ARGB8888 一帧就是 1920×1080×32/8 ≈ **8 MB**，而屏幕是持续刷新的——普通 I/O 要来回拷贝，效率极低。存储映射 I/O 让用户空间直接拿到显存指针，写内存就等于写屏。

```c
screen_size = fb_fix.line_length * fb_var.yres;     // 映射长度
screen_base = mmap(NULL, screen_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
if (MAP_FAILED == screen_base) { perror("mmap error"); ... }
```

- `PROT_WRITE` 必给（要写）；要读回来再或上 `PROT_READ`；
- `MAP_SHARED` 必给，否则改动不回写到设备；
- 长度用 **`line_length × yres`**，不要用 `xres*yres*bpp/8`（行对齐时两者不等）；
- 用完 `munmap(screen_base, screen_size)`。

## 19.4 LCD 基本操作：打点、画线、画矩形

光盘路径：`19_lcd -> lcd_test.c`

### 颜色转换宏（ARGB8888 → RGB565）

```c
#define argb8888_to_rgb565(color) ({                    \
    unsigned int temp = (color);                        \
    ((temp & 0xF80000UL) >> 8) |                        \
    ((temp & 0xFC00UL)   >> 5) |                        \
    ((temp & 0xF8UL)     >> 3); })
```

原理：取 888 的 R 高 5 位、G 高 6 位、B 高 5 位，分别右移到 565 的位置。

### 四个绘图函数

```c
static int width;                          // LCD X 分辨率
static int height;                         // LCD Y 分辨率
static unsigned short *screen_base = NULL; // 映射后的显存基地址
```

**打点**：

```c
static void lcd_draw_point(unsigned int x, unsigned int y, unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);

    if (x >= width)  x = width - 1;
    if (y >= height) y = height - 1;

    screen_base[y * width + x] = rgb565_color;
}
```

**画线**（只支持水平/垂直）：

```c
static void lcd_draw_line(unsigned int x, unsigned int y, int dir,
                          unsigned int length, unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    unsigned int end;
    unsigned long temp;

    if (x >= width)  x = width - 1;
    if (y >= height) y = height - 1;

    temp = y * width + x;               // 定位起点
    if (dir) {                          // 水平线
        end = x + length - 1;
        if (end >= width) end = width - 1;
        for ( ; x <= end; x++, temp++)
            screen_base[temp] = rgb565_color;
    } else {                            // 垂直线
        end = y + length - 1;
        if (end >= height) end = height - 1;
        for ( ; y <= end; y++, temp += width)
            screen_base[temp] = rgb565_color;
    }
}
```

**画矩形**（四条边）：

```c
static void lcd_draw_rectangle(unsigned int start_x, unsigned int end_x,
                               unsigned int start_y, unsigned int end_y,
                               unsigned int color)
{
    int x_len = end_x - start_x + 1;
    int y_len = end_y - start_y - 1;

    lcd_draw_line(start_x, start_y, 1, x_len, color);        // 上边
    lcd_draw_line(start_x, end_y,   1, x_len, color);        // 下边
    lcd_draw_line(start_x, start_y + 1, 0, y_len, color);    // 左边
    lcd_draw_line(end_x,   start_y + 1, 0, y_len, color);    // 右边
}
```

**区域填充**：

```c
static void lcd_fill(unsigned int start_x, unsigned int end_x,
                     unsigned int start_y, unsigned int end_y,
                     unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    unsigned long temp;
    unsigned int x;

    if (end_x >= width)  end_x = width - 1;
    if (end_y >= height) end_y = height - 1;

    temp = start_y * width;                        // 定位到起始行行首
    for ( ; start_y <= end_y; start_y++, temp += width)
        for (x = start_x; x <= end_x; x++)
            screen_base[temp + x] = rgb565_color;
}
```

### main 流程

```c
int main(int argc, char *argv[])
{
    struct fb_fix_screeninfo fb_fix;
    struct fb_var_screeninfo fb_var;
    unsigned int screen_size;
    int fd;

    if (0 > (fd = open("/dev/fb0", O_RDWR))) { perror("open error"); exit(EXIT_FAILURE); }

    ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);

    screen_size = fb_fix.line_length * fb_var.yres;
    width  = fb_var.xres;
    height = fb_var.yres;

    screen_base = mmap(NULL, screen_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (MAP_FAILED == (void *)screen_base) { perror("mmap error"); close(fd); exit(EXIT_FAILURE); }

    /* 画四角方块 + 十字线 + 嵌套矩形 */
    int w = height * 0.25;
    lcd_fill(0, width - 1, 0, height - 1, 0x0);              // 清屏黑
    lcd_fill(0, w, 0, w, 0xFF0000);                          // 红
    lcd_fill(width - w, width - 1, 0, w, 0xFF00);            // 绿
    lcd_fill(0, w, height - w, height - 1, 0xFF);            // 蓝
    lcd_fill(width - w, width - 1, height - w, height - 1, 0xFFFF00); // 黄

    lcd_draw_line(0, height * 0.5, 1, width, 0xFFFFFF);      // 横线
    lcd_draw_line(width * 0.5, 0, 0, height, 0xFFFFFF);      // 竖线

    unsigned int s_x = 0.25 * width, s_y = w;
    unsigned int e_x = width - s_x, e_y = height - s_y;
    for ( ; (s_x <= e_x) && (s_y <= e_y); s_x += 5, s_y += 5, e_x -= 5, e_y -= 5)
        lcd_draw_rectangle(s_x, e_x, s_y, e_y, 0xFFFFFF);

    munmap(screen_base, screen_size);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

编译：

![图19.4.1 编译LCD测试程序](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-2%20%E5%9B%BE19.4.1%20%E7%BC%96%E8%AF%91LCD%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F.png)

拷到板子，**先退出 Qt GUI** 再运行（否则界面会抢 fb 一直重绘）：

![图19.4.2 执行应用程序](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-3%20%E5%9B%BE19.4.2%20%E6%89%A7%E8%A1%8C%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

屏幕效果：黑底 + 四色方块 + 白色十字 + 一圈圈矩形。

![图19.4.3 LCD显示效果](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-4%20%E5%9B%BE19.4.3%20LCD%E6%98%BE%E7%A4%BA%E6%95%88%E6%9E%9C.jpg)

## 19.5 显示 BMP 图片

### BMP 文件结构

BMP（Bitmap）是 Windows 标准位图格式，**不压缩**（所以文件大、无失真、解析简单）。位深支持 1/4/8/16/24/32 bit。

文件由四部分组成：

| 数据段 | 大小 | 说明 |
|---|---|---|
| BMP 文件头 | 14 字节 | 文件类型、文件大小、到位图数据的偏移量 |
| 位图信息头 | 通常 40 或 56 字节 | 信息头大小、图像宽高、图像大小、位平面数、压缩方式、颜色索引 |
| 调色板 | 由颜色索引数决定 | 可选；16/24/32 位真彩色图**没有**调色板 |
| 位图数据 | 由图像尺寸决定 | 真正的像素数据 |

本章用 **16 位（RGB565）BMP**：

![图19.5.1 16位BMP示例](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-5%20%E5%9B%BE19.5.1%2016%E4%BD%8DBMP%E7%A4%BA%E4%BE%8B.jpg)

![图19.5.2 示例图片属性](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-6%20%E5%9B%BE19.5.2%20%E7%A4%BA%E4%BE%8B%E5%9B%BE%E7%89%87%E5%B1%9E%E6%80%A7.png)

### BMP 文件头（14 字节）

```c
typedef struct tagBITMAPFILEHEADER {
    UINT16 bfType;        // 00H，2 字节，必须是 "BM"
    DWORD  bfSize;        // 02H，4 字节，文件总大小
    UINT16 bfReserved1;   // 06H，2 字节，保留，必须 0
    UINT16 bfReserved2;   // 08H，2 字节，保留，必须 0
    DWORD  bfOffBits;     // 0AH，4 字节，从文件头到位图数据的偏移量
} BITMAPFILEHEADER;
```

`bfOffBits` 非常关键：因为信息头长度和调色板会变，**只能靠它定位像素数据**，不能硬编码 54。

十六进制实拍：

![图19.5.3 image.bmp 文件的十六进制数据](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-7%20%E5%9B%BE19.5.3%20image.bmp%20%E6%96%87%E4%BB%B6%E7%9A%84%E5%8D%81%E5%85%AD%E8%BF%9B%E5%88%B6%E6%95%B0%E6%8D%AE.png)

![图19.5.4 bmp 文件头数据](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-8%20%E5%9B%BE19.5.4%20bmp%20%E6%96%87%E4%BB%B6%E5%A4%B4%E6%95%B0%E6%8D%AE.png)

对照：`42 4D` = "BM"；`02~05H` = 0x000BB848 = 768072（与文件大小一致）；`0A~0DH` = 0x46 = 70（像素数据在 70 字节处）。

### 位图信息头

```c
typedef struct tagBITMAPINFOHEADER {
    DWORD biSize;          // 0EH，信息头大小
    LONG  biWidth;         // 12H，图像宽度（像素）
    LONG  biHeight;        // 16H，图像高度；>0 倒向位图，<0 正向位图
    WORD  biPlanes;        // 1AH，总为 1
    WORD  biBitCount;      // 1CH，位深 1/4/8/16/24/32
    DWORD biCompression;   // 1EH，0=RGB，1=RLE8，2=RLE4，3=Bit-Fields
    DWORD biSizeImage;     // 22H，图像数据大小
    LONG  biXPelsPerMeter; // 26H
    LONG  biYPelsPerMeter; // 2AH
    DWORD biClrUsed;       // 2EH，实际使用的颜色索引数
    DWORD biClrImportant;  // 32H
} BITMAPINFOHEADER;
```

![图19.5.5 位图信息头数据](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-9%20%E5%9B%BE19.5.5%20%E4%BD%8D%E5%9B%BE%E4%BF%A1%E6%81%AF%E5%A4%B4%E6%95%B0%E6%8D%AE.png)

实拍解读：`0E~11H` = 0x38 = **56** 字节 → 这是 BF 格式的信息头（比常规 40 字节多 16）；`12~15H` = 800；`16~19H` = 480（正数 → 倒向位图）；`1A~1BH` = 1；`1C~1DH` = 16；`1E~21H` = 3（Bit-Fields）。

> **为什么是 56 而不是 40**：压缩方式填 BF(3) 时，多出的 16 字节是 4 个 32bit 位域掩码（R/G/B/A）。

### 调色板与位图数据

- **调色板**：只有 1/4/8 位（单色、16 色、256 色）图才有；16/24/32 位没有。
- **位图数据**：24 位每像素 3 字节，16 位 2 字节，32 位 4 字节。

### 正向位图 vs 倒向位图

![图19.5.6 正向位图和倒向位图](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-10%20%E5%9B%BE19.5.6%20%E6%AD%A3%E5%90%91%E4%BD%8D%E5%9B%BE%E5%92%8C%E5%80%92%E5%90%91%E4%BD%8D%E5%9B%BE.png)

| 类型 | 判定 | 数据排列 |
|---|---|---|
| 倒向（常见） | `biHeight > 0` | 从**图像左下角**开始，先存最后一行，逐行往**上** |
| 正向 | `biHeight < 0` | 从**左上角**开始，先存第一行，逐行往**下** |

这直接决定了写显存时指针该往上走还是往下走。

### RGB 与 Bit-Fields

16bpp 以上不用调色板，有两种编码：

- **RGB**：均分。24bpp → 低 8 位 B、中 8 位 G、高 8 位 R；32bpp → 再高 8 位是 Alpha。
- **BF（Bit-Fields）**：用位域掩码人为指定各分量占多少位、在什么位置。RGB565 的掩码就是 **R=0xF800、G=0x07E0、B=0x001F**。

![图19.5.7 RGB565位域掩码](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-11%20%E5%9B%BE19.5.7%20RGB565%E4%BD%8D%E5%9F%9F%E6%8E%A9%E7%A0%81.png)

### 怎么得到 RGB565 的 BMP

Windows 下直接另存的多是 24 位 RGB888，喂给本章程序会花屏。原书用 Photoshop：打开图片 → **文件 → 存储为** → 格式选 BMP：

![图19.5.8 设置文件名和文件格式](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-12%20%E5%9B%BE19.5.8%20%E8%AE%BE%E7%BD%AE%E6%96%87%E4%BB%B6%E5%90%8D%E5%92%8C%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F.png)

![图19.5.9 BMP选项](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-13%20%E5%9B%BE19.5.9%20BMP%E9%80%89%E9%A1%B9.png)

选 **16 位色** → 点"高级模式"：

![图19.5.10 BMP高级模式选RGB565](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-14%20%E5%9B%BE19.5.10%20BMP%E9%AB%98%E7%BA%A7%E6%A8%A1%E5%BC%8F%E9%80%89RGB565.png)

选 **R5 G6 B5**（RGB565）→ 确定，即得到 16 位 RGB565 BMP。

> 没有 PS 可以用 ImageMagick：`convert in.jpg -depth 16 -define bmp:subtype=RGB565 out.bmp`

### 例程：bmp_show.c

光盘路径：`19_lcd -> bmp_show.c`

两个结构体必须 `__attribute__((packed))`，否则编译器按 4 字节对齐填充，字段全错位：

```c
/* BMP 文件头 */
typedef struct {
    unsigned char type[2];      // 文件类型
    unsigned int  size;         // 文件大小
    unsigned short reserved1;
    unsigned short reserved2;
    unsigned int  offset;       // 到位图数据的偏移量
} __attribute__((packed)) bmp_file_header;

/* 位图信息头 */
typedef struct {
    unsigned int  size;         // 信息头大小
    int           width;        // 图像宽度
    int           height;       // 图像高度
    unsigned short planes;      // 位面数
    unsigned short bpp;         // 像素深度
    unsigned int  compression;  // 压缩方式
    unsigned int  image_size;   // 图像大小
    int           x_pels_per_meter;
    int           y_pels_per_meter;
    unsigned int  clr_used;
    unsigned int  clr_omportant;
} __attribute__((packed)) bmp_info_header;

static int width, height;
static unsigned short *screen_base = NULL;
static unsigned long line_length;
```

核心函数：

```c
static int show_bmp_image(const char *path)
{
    bmp_file_header file_h;
    bmp_info_header info_h;
    unsigned short *line_buf = NULL;   // 行缓冲
    unsigned long line_bytes;          // BMP 一行的字节数
    unsigned int min_h, min_bytes;
    int fd = -1, j;

    if (0 > (fd = open(path, O_RDONLY))) { perror("open error"); return -1; }

    /* 读文件头 */
    if (sizeof(bmp_file_header) != read(fd, &file_h, sizeof(bmp_file_header))) {
        perror("read error"); close(fd); return -1;
    }
    if (0 != memcmp(file_h.type, "BM", 2)) {
        fprintf(stderr, "it's not a BMP file\n"); close(fd); return -1;
    }

    /* 读信息头 */
    if (sizeof(bmp_info_header) != read(fd, &info_h, sizeof(bmp_info_header))) {
        perror("read error"); close(fd); return -1;
    }

    printf("文件大小: %d\n位图数据的偏移量: %d\n位图信息头大小: %d\n"
           "图像分辨率: %d*%d\n像素深度: %d\n",
           file_h.size, file_h.offset, info_h.size,
           info_h.width, info_h.height, info_h.bpp);

    /* 定位到像素数据起点：必须用 bfOffBits，不能写死 54 */
    if (-1 == lseek(fd, file_h.offset, SEEK_SET)) {
        perror("lseek error"); close(fd); return -1;
    }

    /* 申请一行缓冲 */
    line_bytes = info_h.width * info_h.bpp / 8;
    line_buf = malloc(line_bytes);
    if (NULL == line_buf) { fprintf(stderr, "malloc error\n"); close(fd); return -1; }

    min_bytes = (line_length > line_bytes) ? line_bytes : line_length;

    if (0 < info_h.height) {                       /* 倒向位图 */
        if (info_h.height > height) {
            min_h = height;
            lseek(fd, (info_h.height - height) * line_bytes, SEEK_CUR);
            screen_base += width * (height - 1);   // 定位到屏幕左下角
        } else {
            min_h = info_h.height;
            screen_base += width * (info_h.height - 1);
        }

        for (j = min_h; j > 0; screen_base -= width, j--) {   // 逐行往上
            read(fd, line_buf, line_bytes);
            memcpy(screen_base, line_buf, min_bytes);
        }
    } else {                                        /* 正向位图 */
        int temp = 0 - info_h.height;               // 负数转正
        min_h = (temp > height) ? height : temp;

        for (j = 0; j < min_h; j++, screen_base += width) {    // 逐行往下
            read(fd, line_buf, line_bytes);
            memcpy(screen_base, line_buf, min_bytes);
        }
    }

    close(fd);
    free(line_buf);
    return 0;
}
```

`main`：

```c
int main(int argc, char *argv[])
{
    struct fb_fix_screeninfo fb_fix;
    struct fb_var_screeninfo fb_var;
    unsigned int screen_size;
    int fd;

    if (2 != argc) { fprintf(stderr, "usage: %s <bmp_file>\n", argv[0]); exit(-1); }

    if (0 > (fd = open("/dev/fb0", O_RDWR))) { perror("open error"); exit(EXIT_FAILURE); }

    ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);

    screen_size  = fb_fix.line_length * fb_var.yres;
    line_length  = fb_fix.line_length;
    width        = fb_var.xres;
    height       = fb_var.yres;

    screen_base = mmap(NULL, screen_size, PROT_WRITE, MAP_SHARED, fd, 0);
    if (MAP_FAILED == (void *)screen_base) { perror("mmap error"); close(fd); exit(EXIT_FAILURE); }

    memset(screen_base, 0xFF, screen_size);    // 先刷白，方便看边界
    show_bmp_image(argv[1]);

    munmap(screen_base, screen_size);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

编译、拷程序 + 图片到板子：

![图19.5.11 编译示例代码](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-15%20%E5%9B%BE19.5.11%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

![图19.5.12 将bmp图像和程序拷贝到开发板](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-16%20%E5%9B%BE19.5.12%20%E5%B0%86bmp%E5%9B%BE%E5%83%8F%E5%92%8C%E7%A8%8B%E5%BA%8F%E6%8B%B7%E8%B4%9D%E5%88%B0%E5%BC%80%E5%8F%91%E6%9D%BF.png)

运行（同样先退出 GUI）：

![图19.5.13 执行测试程序](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-17%20%E5%9B%BE19.5.13%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F.png)

LCD 显示结果：

![图19.5.14 LCD显示结果](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-18%20%E5%9B%BE19.5.14%20LCD%E6%98%BE%E7%A4%BA%E7%BB%93%E6%9E%9C.jpg)

原图对照（手机拍照偏色，实际以屏为准）：

![图19.5.15 image.bmp原图](photo/%E7%AC%AC%E5%8D%81%E4%B9%9D%E7%AB%A0%20FrameBuffer%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/19-19%20%E5%9B%BE19.5.15%20image.bmp%E5%8E%9F%E5%9B%BE.jpg)

## 19.6 在你这套环境上实操

**你板子的实际参数**（实测）：`/dev/fb0` 存在，`virtual_size = 1024,600`，`bits_per_pixel = 16`，`stride = 2048`。也就是说 **7 寸 1024×600、RGB565**，`line_length`（2048）恰好等于 `xres×2`，按书上 `y*width+x` 索引不会错。

### 编译（不需要任何库）

```bash
cd ~/linux/IMX6ULL/app-test/19_lcd          # 自己建目录，把光盘 19_lcd 的 .c/.bmp 拷进来
arm-linux-gnueabihf-gcc lcd_info.c -o lcd_info
arm-linux-gnueabihf-gcc lcd_test.c -o lcd_test
arm-linux-gnueabihf-gcc bmp_show.c -o bmp_show
```

> 原书用 `${CC} -o testApp testApp.c`：`${CC}` 是 Yocto SDK 的环境变量（要先 source），`testApp` 是作者统一改的产物名。你直接用 `arm-linux-gnueabihf-gcc` 和原名最省事，**不要三个程序都叫 testApp**，会互相覆盖。

### 部署与运行

```bash
scp lcd_info lcd_test bmp_show image.bmp root@192.168.77.148:/home/root/

# 板子上（注意家目录是 /home/root，不是 /root）
cd ~
chmod +x lcd_info lcd_test bmp_show
./lcd_info                 # 应打印 1024*600 / 16 / 2048 / R<11 5> G<5 6> B<0 5>
./lcd_test                 # 四色方块 + 十字线 + 嵌套矩形
./bmp_show image.bmp       # 显示图片
```

`image.bmp` 是 800×480 的，比屏幕小，会画在左下角（倒向位图从下往上）；想做满屏就换一张 1024×600 的 RGB565 BMP。

### 踩坑对照表

| 现象 | 原因 | 处理 |
|---|---|---|
| 程序跑完立刻恢复原界面 | Qt GUI 还在刷 fb | 先退出出厂 GUI 再运行 |
| 全黑但程序返回 0 | 没映射成功或没写进去 | `dd if=/dev/zero of=/dev/fb0` 验证链路；检查 `mmap` 返回值 |
| 颜色偏（红蓝互换） | 用了 888 值没转 565，或屏是 BGR | 用 `argb8888_to_rgb565`；看 `fb_var.red/green/blue.offset` |
| 图像斜着错位/花屏 | `line_length ≠ xres*bpp/8` | 索引改用 `line_length/2` 作为"每行多少 short" |
| BMP 显示一半乱码 | BMP 不是 RGB565，或没按 `bfOffBits` 跳 | 重新导出 16 位 RGB565；`lseek` 用 `file_h.offset` |
| 图像上下颠倒 | `biHeight` 正负判反 | 正数=倒向（从下往上写），负数=正向 |
| `fatal error: linux/fb.h` | 交叉编译器 sysroot 缺头文件 | 确认用的是 `arm-linux-gnueabihf-gcc`（自带内核头） |
| 编译报 `undefined reference` | 误加了 `-lxxx` | 本章不需要任何 `-l` |

## 本章速记

| 要点 | 内容 |
|---|---|
| FrameBuffer 是什么 | 对显示设备的抽象，一块保存一帧图像的内存；`/dev/fb0`~`/dev/fb31` |
| 显存大小 | `xres × yres × bpp / 8`；映射长度用 `line_length × yres` |
| 五步流程 | open → ioctl 取参数 → mmap → 读写显存 → munmap + close |
| 三个 ioctl | `FBIOGET_VSCREENINFO`(0x4600) / `FBIOPUT_VSCREENINFO`(0x4601) / `FBIOGET_FSCREENINFO`(0x4602) |
| var 关键字段 | `xres`、`yres`、`bits_per_pixel`、`red/green/blue`(offset+length) |
| fix 关键字段 | `line_length`（一行字节数）、`smem_start`、`smem_len` |
| 判断像素格式 | `R<11 5> G<5 6> B<0 5>` = RGB565；`R<16 8>` = RGB888 |
| mmap 参数 | `PROT_READ|PROT_WRITE`、`MAP_SHARED`、偏移 0；用 `MAP_FAILED` 判失败 |
| 打点公式 | `screen_base[y*width + x]`（16bpp，指针为 `unsigned short *`） |
| 与 tslib 的区别 | fb 是内核驱动，**不需要移植任何库**；tslib/libjpeg/libpng/freetype 才是用户态库 |
| BMP 四段 | 文件头 14B + 信息头 40/56B + 调色板(可选) + 位图数据 |
| 定位像素数据 | 只用 `bfOffBits`（`lseek`），不要硬编码 54 |
| 倒向位图 | `biHeight > 0`，数据从图像左下角开始，写屏时指针**往上**移 |
| RGB565 掩码 | R=0xF800、G=0x07E0、B=0x001F；压缩方式填 3(BF) 时信息头为 56 字节 |
| 结构体对齐 | 解析文件头的结构体必须 `__attribute__((packed))` |
| 测试前提 | 退出 Qt GUI，否则界面持续重绘覆盖你的画面 |
