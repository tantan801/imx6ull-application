# 提高篇第二十二章 LCD 横屏切换为竖屏

> 配套例程：`22_lcd_vertical_display/lcd_vertical_display.c`
> 一句话：**横竖屏切换跟驱动无关，是应用层的事**。驱动/硬件只认"左上角为 (0,0)、从左到右、从上到下"的横屏坐标；竖屏是应用在打点时自己做一次坐标旋转。

本章主题：
- 横屏显示如何切换为竖屏显示
- 编写代码验证

---

## 22.1 横屏显示如何切换为竖屏显示

### 硬件层：屏幕天生是横屏

正点原子配套的 RGB LCD 都是横屏：4.3 寸 480×272、4.3 寸 800×480、7 寸 800×480、7 寸 1024×600、10.1 寸 1280×800 等。

以 800×480 为例，屏幕正向放置时，**硬件固定**的坐标是：

- 左上角 `(0, 0)`
- 左下角 `(0, 480-1)`
- 右上角 `(800-1, 0)`
- 右下角 `(800-1, 480-1)`

![22-1 图22.1.1 LCD 屏正向放置](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%BA%8C%E7%AB%A0%20LCD%E6%A8%AA%E5%B1%8F%E5%88%87%E6%8D%A2%E4%B8%BA%E7%AB%96%E5%B1%8F/22-1%20%E5%9B%BE22.1.1%20LCD%20%E5%B1%8F%E6%AD%A3%E5%90%91%E6%94%BE%E7%BD%AE.png)

这是**不可修改的硬件属性**：你没法把硬件配置成"左下角是 (0,0)"。像素排列永远是**从左到右、从上到下**，操作某个像素就是算它对应的显存地址。

### 显存地址公式（横屏/物理坐标）

设显存基地址为 `base`（`unsigned char *`），像素 (x, y) 的地址：

```
base + (y * width + x) * pix_bytes
```

- `width`：LCD 横向分辨率
- `pix_bytes`：一个像素占几个字节（RGB565 为 2）

![22-2 图22.1.2 LCD 像素点与显存对应关系示意图](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%BA%8C%E7%AB%A0%20LCD%E6%A8%AA%E5%B1%8F%E5%88%87%E6%8D%A2%E4%B8%BA%E7%AB%96%E5%B1%8F/22-2%20%E5%9B%BE22.1.2%20LCD%20%E5%83%8F%E7%B4%A0%E7%82%B9%E4%B8%8E%E6%98%BE%E5%AD%98%E5%AF%B9%E5%BA%94%E5%85%B3%E7%B3%BB%E7%A4%BA%E6%84%8F.png)

### 应用层怎么做竖屏

很多场景要竖着看（手机就是竖屏）。竖屏的本质：**应用程序自己定义一套坐标**，打点时把它换算成物理坐标再写显存。

以 800×480 横屏为例，若规定"**左下角为竖屏起点 (0,0)**"，则竖屏坐标对应：

- 起点 `(0, 0)` → 物理 `(0, 480-1)`（左下角）
- 左上 `(480-1, 0)`
- 右下 `(0, 800-1)`
- 右上 `(480-1, 800-1)`

![22-3 图22.1.3 竖屏方式坐标分布](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%BA%8C%E7%AB%A0%20LCD%E6%A8%AA%E5%B1%8F%E5%88%87%E6%8D%A2%E4%B8%BA%E7%AB%96%E5%B1%8F/22-3%20%E5%9B%BE22.1.3%20%E7%AB%96%E5%B1%8F%E6%96%B9%E5%BC%8F%E5%9D%90%E6%A0%87%E5%88%86%E5%B8%83.jpg)

**注意**：这只是"应用程序认为的坐标"，LCD 硬件实际起点仍是左上角。

### 竖屏坐标 → 显存地址公式

竖屏 (x, y) 对应的物理坐标是：

```
x_phys = y
y_phys = height - 1 - x
```

代入物理公式，得到**本书这种竖屏**的映射：

```
base + ((height - 1 - x) * width + y) * pix_bytes
```

- x、y 是**竖屏坐标**；
- `height` 是物理屏高（横屏时的 yres），`width` 是物理屏宽（xres）。

> 这个公式**只适用于上面这种"左下角为原点"的竖屏**。如果把画面再旋转 180° 倒过来，也是竖屏，但公式就变了（自己按同样思路推即可，本质就是坐标变换）。
>
> 代码里常用 `lcd_max_y = lcd_height - 1` 来表示 `height - 1`。

---

## 22.2 编写示例代码

基于示例代码 19.4.1（LCD 画图）修改成竖屏。源码：`22_lcd_vertical_display/lcd_vertical_display.c`。

```c
/***************************************************************
 Copyright © ALIENTEK Co., Ltd. 1998-2021. All rights reserved.
 文件名 : lcd_vertical_display.c
 作者 : 邓涛
 版本 : V1.0
 描述 : FrameBuffer 应用编程之横屏切换为竖屏显示
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
#include <sys/mman.h>
#include <linux/fb.h>

#define argb8888_to_rgb565(color) ({ \
    unsigned int temp = (color); \
    ((temp & 0xF80000UL) >> 8) | \
    ((temp & 0xFC00UL) >> 5) | \
    ((temp & 0xF8UL) >> 3); \
})

static int lcd_width;                      // LCD X 分辨率
static int lcd_height;                     // LCD Y 分辨率
static int lcd_max_y;                      // LCD Y 坐标最大值
static int user_width;                     // 竖屏模式下 X 分辨率
static int user_height;                    // 竖屏模式下 Y 分辨率
static unsigned short *screen_base = NULL; // 映射后的显存基地址

/* 打点 */
static void lcd_draw_point(unsigned int x, unsigned int y, unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);

    if (x >= user_width)
        x = user_width - 1;
    if (y >= user_height)
        y = user_height - 1;

    /* 竖屏坐标 -> 物理显存地址 */
    screen_base[(lcd_max_y - x) * lcd_width + y] = rgb565_color;
}

/* 画线（dir=1 水平线，dir=0 垂直线） */
static void lcd_draw_line(unsigned int x, unsigned int y, int dir,
                          unsigned int length, unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    unsigned int end;
    unsigned long temp;

    if (x >= user_width)
        x = user_width - 1;
    if (y >= user_height)
        y = user_height - 1;

    temp = (lcd_max_y - x) * lcd_width + y;
    if (dir) {
        /* 水平线 */
        end = x + length - 1;
        if (end >= user_width)
            end = user_width - 1;
        for ( ; x <= end; x++, temp -= lcd_width)
            screen_base[temp] = rgb565_color;
    } else {
        /* 垂直线 */
        end = y + length - 1;
        if (end >= user_height)
            end = user_height - 1;
        for ( ; y <= end; y++, temp++)
            screen_base[temp] = rgb565_color;
    }
}

/* 画矩形 */
static void lcd_draw_rectangle(unsigned int start_x, unsigned int end_x,
                               unsigned int start_y, unsigned int end_y,
                               unsigned int color)
{
    int x_len = end_x - start_x + 1;
    int y_len = end_y - start_y - 1;

    lcd_draw_line(start_x, start_y, 1, x_len, color);       // 上边
    lcd_draw_line(start_x, end_y, 1, x_len, color);         // 下边
    lcd_draw_line(start_x, start_y + 1, 0, y_len, color);   // 左边
    lcd_draw_line(end_x, start_y + 1, 0, y_len, color);     // 右边
}

/* 填充矩形区域 */
static void lcd_fill(unsigned int start_x, unsigned int end_x,
                     unsigned int start_y, unsigned int end_y,
                     unsigned int color)
{
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    unsigned long temp;
    unsigned long step_size_count;
    int x;

    if (end_x >= user_width)
        end_x = user_width - 1;
    if (end_y >= user_height)
        end_y = user_height - 1;

    temp = (lcd_max_y - start_x) * lcd_width + start_y;
    for ( ; start_y <= end_y; start_y++, temp++) {
        step_size_count = 0;
        for (x = start_x; x <= end_x; x++, step_size_count += lcd_width)
            screen_base[temp - step_size_count] = rgb565_color;
    }
}

int main(int argc, char *argv[])
{
    struct fb_fix_screeninfo fb_fix;
    struct fb_var_screeninfo fb_var;
    unsigned int screen_size;
    int fd;

    if (0 > (fd = open("/dev/fb0", O_RDWR))) {
        perror("open error");
        exit(EXIT_FAILURE);
    }

    ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);

    screen_size = fb_fix.line_length * fb_var.yres;
    lcd_width = fb_var.xres;
    lcd_height = fb_var.yres;
    lcd_max_y = lcd_height - 1;
    user_width = fb_var.yres;      // 竖屏宽 = 物理高
    user_height = fb_var.xres;     // 竖屏高 = 物理宽

    screen_base = mmap(NULL, screen_size, PROT_WRITE, MAP_SHARED, fd, 0);
    if (MAP_FAILED == (void *)screen_base) {
        perror("mmap error");
        close(fd);
        exit(EXIT_FAILURE);
    }

    /* 画正方形方块 */
    int w = user_height * 0.25;    // 1/4 屏幕高度
    lcd_fill(0, user_width - 1, 0, user_height - 1, 0x0);              // 清屏（黑）
    lcd_fill(0, w, 0, w, 0xFF0000);                                    // 红色方块
    lcd_fill(user_width - w, user_width - 1, 0, w, 0xFF00);            // 绿色方块
    lcd_fill(0, w, user_height - w, user_height - 1, 0xFF);            // 蓝色方块
    lcd_fill(user_width - w, user_width - 1,
             user_height - w, user_height - 1, 0xFFFF00);              // 黄色方块

    /* 十字交叉线 */
    lcd_draw_line(0, user_height * 0.5, 1, user_width, 0xFFFFFF);      // 白色水平线
    lcd_draw_line(user_width * 0.5, 0, 0, user_height, 0xFFFFFF);      // 白色垂直线

    /* 嵌套矩形 */
    unsigned int s_x, s_y, e_x, e_y;
    s_x = 0.25 * user_width;
    s_y = w;
    e_x = user_width - s_x;
    e_y = user_height - s_y;

    for ( ; (s_x <= e_x) && (s_y <= e_y); s_x += 5, s_y += 5, e_x -= 5, e_y -= 5)
        lcd_draw_rectangle(s_x, e_x, s_y, e_y, 0xFFFFFF);

    munmap(screen_base, screen_size);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

### 代码关键点

1. **`user_width` / `user_height` 互换**：
   ```c
   user_width  = fb_var.yres;   // 竖屏宽 = 物理高（600）
   user_height = fb_var.xres;   // 竖屏高 = 物理宽（1024）
   ```
   这是"竖屏"的入口，之后所有绘图函数都用 `user_*` 做边界检查。
2. **4 个绘图函数全部基于竖屏坐标**：调用者传的是竖屏 (x, y)，函数内部用 `(lcd_max_y - x) * lcd_width + y` 换成物理地址。
3. **`lcd_max_y = lcd_height - 1`**，对应公式里的 `height - 1`。
4. **水平线/垂直线在竖屏下方向也变了**：`dir=1` 的"水平线"在物理上是沿 y 轴移动（`temp -= lcd_width`），因为竖屏 x 方向对应物理 y 方向。
5. `argb8888_to_rgb565` 是 GNU 语句表达式宏（`({ ... })`），gcc 支持。

### 编译（不需要额外库）

```bash
${CC} -o lcd_vertical_display lcd_vertical_display.c
```

> 纯 framebuffer 操作，不链接第三方库。

### 运行

```bash
scp lcd_vertical_display root@<板子IP>:/home/root/
ssh root@<板子IP>
cd /home/root
./lcd_vertical_display
```

执行前关掉出厂 Qt GUI，避免抢显存。

![22-4 图22.2.1 LCD 显示效果](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%BA%8C%E7%AB%A0%20LCD%E6%A8%AA%E5%B1%8F%E5%88%87%E6%8D%A2%E4%B8%BA%E7%AB%96%E5%B1%8F/22-4%20%E5%9B%BE22.2.1%20LCD%20%E6%98%BE%E7%A4%BA%E6%95%88%E6%9E%9C.jpg)

和第 19 章图 19.4.3 对比：同样的画面，一个是横着显示、一个是竖着显示。

---

## 实测记录（ATK-IMX6U 实机）

- 板子屏幕：**1024×600，16bpp，stride=2048**。
- 编译：`${CC} -o lcd_vertical_display lcd_vertical_display.c` → `rc=0`，`ELF 32-bit ARM`。
- 运行：`./lcd_vertical_display` → **rc=0**，LCD 竖屏显示 4 个角块 + 十字线 + 嵌套矩形。
- 显存回读（1,228,800 字节 = 1024×600×2）做像素统计验证：

| 颜色 | RGB565 | 像素数 | 预期 |
|------|--------|--------|------|
| 红 | `0xF800` | 65942 | ≈256×257 |
| 绿 | `0x07E0` | 65685 | ≈256×257 |
| 蓝 | `0x001F` | 65685 | ≈256×257 |
| 黄 | `0xFFE0` | 65429 | ≈256×257 |
| 白 | `0xFFFF` | 32823 | 十字线+矩形 |
| 黑 | `0x0000` | 318836 | 背景 |

- 4 个关键点抽查，竖屏坐标 → 显存索引 → 颜色全部正确：
  - `RED(x=128,y=128)` → `F800`
  - `GREEN(x=471,y=128)` → `07E0`
  - `BLUE(x=128,y=896)` → `001F`
  - `YELLOW(x=471,y=896)` → `FFE0`
- 结论：竖屏坐标映射公式 `(lcd_max_y - x) * lcd_width + y` 在 1024×600 上实测正确。
