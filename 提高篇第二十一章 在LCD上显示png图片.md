# 提高篇第二十一章 在 LCD 上显示 png 图片

> 配套例程：`21_libpng/show_png_image.c`
> 核心：libpng 依赖 zlib，所以**先移植 zlib、再移植 libpng**；用 libpng 解出 RGB888，转 RGB565 刷显存。
> 上一章用 libjpeg 显示 jpg，本章换成 png，显示流程（framebuffer 那套）完全一样，换的是"解码库 + 数据格式"。

本章主题：
- PNG 简介
- libpng 库简介
- libpng 库移植（zlib + libpng）
- 用 libpng 库函数解码 PNG

---

## 21.1 PNG 简介

**PNG**（Portable Network Graphic Format，便携式网络图形格式）是一种采用**无损压缩**算法的位图格式，设计目的是替代 GIF 和 TIFF 并增加 GIF 没有的特性。

它使用从 **LZ77 派生**的无损压缩算法，压缩比高、文件体积小、支持透明。

**五大特点：**

| 特点 | 说明 |
|------|------|
| 无损压缩 | 用 LZ77 派生算法，压缩比高且**不损失数据**，重复保存不降低画质 |
| 体积小 | 在清晰、逼真、不失真前提下，压缩比高、文件小 |
| 索引彩色模式 | PNG-8 类似 GIF，用 8 位调色板把 RGB 转成索引彩色，数据量更小 |
| 网络传输优化 | 浏览器流式浏览，隔行处理后未下完也能先显示轮廓再变清晰 |
| 支持透明 | 可定义 256 个透明层次，边缘能平滑融合背景，消除锯齿（GIF/JPEG 没有） |

对比 JPEG：PNG **无损**、支持透明；JPEG **有损**、压缩比更高。选哪个看需求。

---

## 21.2 libpng 简介

- **libpng** 是免费、开源的 **C 语言函数库**，用于 png 图像的**解码、编码**等功能。
- 和 libjpeg 一样，属于"别人写好的轮子"，直接调用即可。
- **重要依赖**：libpng 依赖 **zlib**（zlib 提供 Deflate 压缩/解压）→ 必须**先移植 zlib**。

---

## 21.3 zlib 移植

zlib 是一套包含数据压缩算法的函数库，自由软件、免费开源。

> 出厂系统其实已经移植好 zlib/libpng，可直接用；但学习阶段必须自己亲自移植一遍。

### 21.3.1 下载源码包

到 <https://www.zlib.net/fossils/> 下载：

![21-1 图21.3.1 zlib 源码下载链接](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-1%20%E5%9B%BE21.3.1%20zlib%20%E6%BA%90%E7%A0%81%E4%B8%8B%E8%BD%BD%E9%93%BE%E6%8E%A5%20%E5%BE%80%E4%B8%8B%E7%BF%BB%E6%89%BE%E5%88%B0%E4%B8%80%E4%B8%AA%E5%90%88%E9%80%82%E7%9A%84%E7%89%88%E6%9C%AC.png)

书里选 **1.2.10** 版本：

![21-2 图21.3.2 选择一个版本](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-2%20%E5%9B%BE21.3.2%20%E9%80%89%E6%8B%A9%E4%B8%80%E4%B8%AA%E7%89%88%E6%9C%AC%20%E7%82%B9%E5%87%BB%E6%96%87%E4%BB%B6%E5%90%8D%E5%B0%B1.png)

得到 `zlib-1.2.10.tar.gz`。

### 21.3.2 编译源码

拷到 Ubuntu 家目录并解压：

```bash
tar -xzf zlib-1.2.10.tar.gz
```

得到 `zlib-1.2.10` 源码目录，里面是 zlib 源码：

![21-5 图21.3.6 zlib 源码目录下的文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-5%20%E5%9B%BE21.3.6%20zlib%20%E6%BA%90%E7%A0%81%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%20%E5%90%8C%E6%A0%B7%E4%B9%9F%E6%98%AF%E6%89%A7%E8%A1%8C%E4%B8%89%E9%83%A8%E6%9B%B2%E9%85%8D.png)

![21-3 图21.3.4 将 zlib 压缩文件解压](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-3%20%E5%9B%BE21.3.4%20%E5%B0%86%20zlib%20%E5%8E%8B%E7%BC%A9%E6%96%87%E4%BB%B6%E8%A7%A3%E5%8E%8B%20%E8%A7%A3%E5%8E%8B%E4%B9%8B%E5%90%8E%E5%B0%B1%E4%BC%9A%E5%BE%97%E5%88%B0%20z.png)

在 `~/tools` 下建安装目录 `zlib`：

![21-4 图21.3.5 创建 zlib 文件夹](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-4%20%E5%9B%BE21.3.5%20%E5%88%9B%E5%BB%BA%20zlib%20%E6%96%87%E4%BB%B6%E5%A4%B9%20%E6%8E%A5%E7%9D%80.png)

```bash
mkdir -p ~/tools/zlib
```

进入源码目录，还是**配置 → 编译 → 安装**。先初始化交叉编译环境：

```bash
source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
```

配置：

```bash
./configure --prefix=/home/dt/tools/zlib/
```

- `--prefix`：zlib 安装目录。
- 原书这里**没写 `--host`**，靠已初始化的环境变量里的交叉编译器。**稳妥做法**：显式加上 `--host=arm-poky-linux-gnueabi`（或用你自己的前缀），避免误用本机 gcc。

![21-6 图21.3.7 配置 zlib 源码工程](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-6%20%E5%9B%BE21.3.7%20%E9%85%8D%E7%BD%AE%20zlib%20%E6%BA%90%E7%A0%81%E5%B7%A5%E7%A8%8B.png)

编译 + 安装：

```bash
make
make install
```

![21-7 图21.3.8 make 编译 zlib](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-7%20%E5%9B%BE21.3.8%20make%20%E7%BC%96%E8%AF%91%20zlib%20%E7%BC%96%E8%AF%91%E5%AE%8C%E6%88%90%E4%B9%8B%E5%90%8E%E6%8E%A5%E7%9D%80%E6%89%A7%E8%A1%8C.png)

### 21.3.3 安装目录与部署

安装目录只有 `include` 和 `lib` 两个目录：

![21-8 图21.3.10 zlib 安装目录下的文件夹](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-8%20%E5%9B%BE21.3.10%20zlib%20%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%E5%A4%B9%20%E5%A4%B4%E6%96%87%E4%BB%B6%E7%9B%AE%E5%BD%95%20inc.png)

部署到板子：把 `lib` 下的动态库拷到板子 `/usr/lib`。拷之前先删出厂旧库：

```bash
rm -rf /usr/lib/libz.* /lib/libz.*
```

再传新库，**注意符号链接不要破坏**（打包再解压，或 `cp -a`）：

![21-9 图21.3.11 开发板 /usr/lib 目录下的 zlib 库文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-9%20%E5%9B%BE21.3.11%20%E5%BC%80%E5%8F%91%E6%9D%BFusrlib%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%20zlib%20%E5%BA%93%E6%96%87%E4%BB%B6.png)

> **和 libjpeg 同样的坑**：替换 `/usr/lib` 里的系统库可能影响出厂 Qt 等程序。稳妥做法是装到独立目录（如 `/home/root/png_lib`），运行时 `LD_LIBRARY_PATH` 指过去。zlib 是很多程序的公共依赖，**能不覆盖系统库就别覆盖**。

---

## 21.4 libpng 移植

zlib 好了，接着移植 libpng。

### 21.4.1 下载源码包

到 <https://github.com/glennrp/libpng/releases> 下载：

![21-10 图21.4.1 libpng 源码下载链接](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-10%20%E5%9B%BE21.4.1%20libpng%20%E6%BA%90%E7%A0%81%E4%B8%8B%E8%BD%BD%E9%93%BE%E6%8E%A5.png)

书里用 **1.6.35**，点 `Source Code(tar.gz)` 下载：

![21-11 图21.4.2 选择 1.6.35 版本 tar.gz](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-11%20%E5%9B%BE21.4.2%20%E9%80%89%E6%8B%A9%201.6.35%20%E7%89%88%E6%9C%AC%20tar.gz%20%E5%8E%8B%E7%BC%A9%E6%96%87%E4%BB%B6.png)

![21-12 图21.4.3 libpng 源码包](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-12%20%E5%9B%BE21.4.3%20libpng%20%E6%BA%90%E7%A0%81%E5%8C%85.png)

### 21.4.2 编译源码

解压：

```bash
tar -xzf libpng-1.6.35.tar.gz
```

得到 `libpng-1.6.35` 源码目录。

![21-13 图21.4.4 将 libpng-1.6.35.tar.gz 解压](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-13%20%E5%9B%BE21.4.4%20%E5%B0%86%20libpng-1.6.35.tar.gz%20%E8%A7%A3.png)

在 `~/tools` 下建安装目录 `png`：

![21-14 图21.4.5 创建 png 目录](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-14%20%E5%9B%BE21.4.5%20%E5%88%9B%E5%BB%BA%20png%20%E7%9B%AE%E5%BD%95%20%E6%8E%A5%E7%9D%80.png)

```bash
mkdir -p ~/tools/png
```

初始化交叉编译环境：

```bash
source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
```

**关键**：libpng 依赖 zlib，必须告诉编译器 zlib 的头文件和库在哪，否则编译报错。导出环境变量：

```bash
export LDFLAGS="${LDFLAGS} -L/home/dt/tools/zlib/lib"
export CFLAGS="${CFLAGS} -I/home/dt/tools/zlib/include"
export CPPFLAGS="${CPPFLAGS} -I/home/dt/tools/zlib/include"
```

![21-15 图21.4.6 导出环境变量](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-15%20%E5%9B%BE21.4.6%20%E5%AF%BC%E5%87%BA%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%20%E6%8E%A5%E7%9D%80%E6%89%A7%E8%A1%8C.png)

配置：

```bash
./configure --prefix=/home/dt/tools/png --host=arm-poky-linux-gnueabi
```

![21-16 图21.4.7 配置 libpng 源码工程](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-16%20%E5%9B%BE21.4.7%20%E9%85%8D%E7%BD%AE%20libpng%20%E6%BA%90%E7%A0%81%E5%B7%A5%E7%A8%8B%20%E6%8E%A5%E7%9D%80%E6%89%A7%E8%A1%8C%20make.png)

编译 + 安装：

```bash
make
make install
```

![21-17 图21.4.8 make 编译](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-17%20%E5%9B%BE21.4.8%20make%20%E7%BC%96%E8%AF%91%20%E6%9C%80%E5%90%8E%E6%89%A7%E8%A1%8C%20make%20instal.png)

![21-18 图21.4.9 make install](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-18%20%E5%9B%BE21.4.9%20make%20install%20%E5%AE%89%E8%A3%85.png)

### 21.4.3 安装目录与部署

安装目录同样有 `bin`、`include`、`lib`：

![21-19 图21.4.10 libpng 安装目录下的文件夹](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-19%20%E5%9B%BE21.4.10%20libpng%20%E5%AE%89%E8%A3%85%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E6%96%87%E4%BB%B6%E5%A4%B9%20%E5%90%8C%E6%A0%B7%E5%8C%85%E5%90%AB%E4%BA%86%20b.png)

部署：`bin` 下测试工具 → `/usr/bin`；`lib` 下库文件 → `/usr/lib`。先删出厂旧库：

```bash
rm -rf /lib/libpng* /usr/lib/libpng*
```

再拷新库，注意符号链接。

![21-20 图21.4.11 开发板 /usr/lib 目录下的 libpng 相关库文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-20%20%E5%9B%BE21.4.11%20%E5%BC%80%E5%8F%91%E6%9D%BFusrlib%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%20libpng%20%E7%9B%B8%E5%85%B3.png)

---

## 21.5 libpng 使用说明

应用里 `#include <png.h>`。

官方手册（内容相同，pdf / txt 二选一）：
- <http://www.libpng.org/pub/png/libpng-1.4.0-manual.pdf>
- <http://www.libpng.org/pub/png/libpng-manual.txt>

### 21.5.1 两个核心数据结构

| 结构体 | 作用 |
|--------|------|
| `png_struct` | libpng 内部使用，作为每个库函数的**第一个参数**；用户一般不直接读其成员 |
| `png_info` | 描述 png 图像信息（宽、高、位深、颜色类型等）。新版通过 `png_get_XXX` / `png_set_XXX` 访问，不建议直接改成员 |

### 21.5.2 创建并初始化 png_struct

```c
png_structp png_create_read_struct(png_const_charp user_png_ver,
                                   png_voidp error_ptr,
                                   png_error_ptr error_fn,
                                   png_error_ptr warn_fn);
```

- 返回 `png_structp`；失败返回 `NULL`。
- `user_png_ver`：版本字符串，固定填宏 `PNG_LIBPNG_VER_STRING`（如 `"1.6.35"`）。
- `error_fn` / `warn_fn`：自定义错误 / 警告函数；填 `NULL` 用默认。
- `error_ptr`：传给上述函数的数据指针。

```c
png_structp png_ptr = NULL;
png_ptr = png_create_read_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
if (!png_ptr)
    return -1;
```

编码方向的对应函数是 `png_create_write_struct()`。

### 21.5.3 创建并初始化 png_info

```c
png_infop png_create_info_struct(png_const_structrp png_ptr);
```

- 参数是刚创建的 `png_struct` 指针，内部建立关联。
- 返回 `png_infop`；失败返回 `NULL`。

```c
png_infop info_ptr = NULL;
info_ptr = png_create_info_struct(png_ptr);
if (NULL == info_ptr) {
    png_destroy_read_struct(&png_ptr, NULL, NULL);
    return -1;
}
```

### 21.5.4 设置错误返回点（setjmp / longjmp）

**为什么要这一步：** libpng 默认错误处理会 `longjmp` 跳转，不能让它直接把进程干掉，得跳回我们程序里做清理（销毁 png_struct / png_info、关文件），避免内存泄漏。

**setjmp / longjmp 原理（跨函数跳转）：**

- `goto` 只能函数内跳；`setjmp` / `longjmp` 可**跨函数跳转**。
- `#include <setjmp.h>`
  ```c
  int  setjmp(jmp_buf env);       // 设置跳转点，保存当前环境
  void longjmp(jmp_buf env, int val); // 跳到 setjmp 处
  ```
- `setjmp()` 首次调用返回 **0**；被 `longjmp()` 跳回后，`setjmp()` 再次"返回"，返回值 = `longjmp` 的 `val`。
- **`val` 不能设 0**，否则无法区分"首次返回"和"跳回"。

示例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <setjmp.h>

static jmp_buf buf;

static void hello(void)
{
    printf("hello world!\n");
    longjmp(buf, 1);
    printf("Nice to meet you!\n");   // 永远不会执行
}

int main(void)
{
    if (0 == setjmp(buf)) {
        printf("First return\n");
        hello();
    } else
        printf("Second return\n");

    exit(0);
}
```

![21-21 图21.5.1 测试程序运行结果](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-21%20%E5%9B%BE21.5.1%20%E6%B5%8B%E8%AF%95.png)

**libpng 设置返回点：**

```c
/* 设置错误返回点 */
if (setjmp(png_jmpbuf(png_ptr))) {
    png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
    return -1;
}
```

`png_jmpbuf(png_ptr)` 取出 `png_struct` 里的 `jmp_buf`，libpng 出错 `longjmp` 时用的就是它。跳回后在这里做清理。

### 21.5.5 指定数据源

用文件输入流：

```c
png_init_io(png_structrp png_ptr, png_FILE_p fp);
```

第二个参数就是标准 I/O 的 `FILE *`。先 `fopen` 再传入：

```c
FILE *png_file = NULL;
png_file = fopen("image.png", "r");
if (NULL == png_file) {
    perror("fopen error");
    return -1;
}
png_init_io(png_ptr, png_file);
```

数据源也可以是内存，不限于文件。

### 21.5.6 读取并解码

两种方式：**high-level**（一个函数搞定，简单但不灵活）、**low-level**（多函数，灵活）。

#### high-level 接口

适用条件：
- 内存足够，能一次性放下整张图解码后的数据；
- 输出格式限定为 libpng 预定义转换类型。

**预定义转换类型表：**

| 转换类型 | 说明 |
|----------|------|
| `PNG_TRANSFORM_IDENTITY` | 不转换 |
| `PNG_TRANSFORM_STRIP_16` | 16 位样本降为 8 位 |
| `PNG_TRANSFORM_STRIP_ALPHA` | 丢弃 alpha 通道 |
| `PNG_TRANSFORM_PACKING` | 1/2/4 位样本展开为字节 |
| `PNG_TRANSFORM_PACKSWAP` | 打包像素改为 LSB 优先 |
| `PNG_TRANSFORM_EXPAND` | 执行 set_expand() |
| `PNG_TRANSFORM_INVERT_MONO` | 单色反相 |
| `PNG_TRANSFORM_SHIFT` | 归一化到 sBIT 深度 |
| `PNG_TRANSFORM_BGR` | RGB→BGR、RGBA→BGRA |
| `PNG_TRANSFORM_SWAP_ALPHA` | RGBA→ARGB、GA→AG |
| `PNG_TRANSFORM_INVERT_ALPHA` | alpha 由不透明改为透明 |
| `PNG_TRANSFORM_SWAP_ENDIAN` | 16 位样本字节交换 |
| `PNG_TRANSFORM_GRAY_TO_RGB` | 灰度扩展为 RGB |

可多个用 `|` 组合。

```c
png_read_png(png_structrp png_ptr, png_inforp info_ptr,
             int transforms, png_voidp params);
```

一次性解码整张图，内部相当于：

1. `png_read_info` 获取图像信息；
2. 按 `transforms` 设置输出转换；
3. `png_read_image` 解码整图到内存；
4. `png_read_end` 结束解码。

```c
png_read_png(png_ptr, info_ptr, PNG_TRANSFORM_STRIP_ALPHA, NULL);
```

#### low-level 接口（分步）

**a) 读信息**

```c
png_read_info(png_ptr, info_ptr);
```

**b) 查信息**

```c
unsigned int  width      = png_get_image_width(png_ptr, info_ptr);  // 宽
unsigned int  height     = png_get_image_height(png_ptr, info_ptr); // 高
unsigned char depth      = png_get_bit_depth(png_ptr, info_ptr);    // 位深
unsigned char color_type = png_get_color_type(png_ptr, info_ptr);   // 颜色类型
```

颜色类型宏：

```c
#define PNG_COLOR_TYPE_GRAY        0
#define PNG_COLOR_TYPE_PALETTE     (PNG_COLOR_MASK_COLOR | PNG_COLOR_MASK_PALETTE)
#define PNG_COLOR_TYPE_RGB         (PNG_COLOR_MASK_COLOR)
#define PNG_COLOR_TYPE_RGB_ALPHA   (PNG_COLOR_MASK_COLOR | PNG_COLOR_MASK_ALPHA)
#define PNG_COLOR_TYPE_GRAY_ALPHA  (PNG_COLOR_MASK_ALPHA)
/* 别名 */
#define PNG_COLOR_TYPE_RGBA        PNG_COLOR_TYPE_RGB_ALPHA
#define PNG_COLOR_TYPE_GA          PNG_COLOR_TYPE_GRAY_ALPHA

/* 掩码 */
#define PNG_COLOR_MASK_PALETTE     1
#define PNG_COLOR_MASK_COLOR       2
#define PNG_COLOR_MASK_ALPHA       4
```

**c) 设置输出转换参数（重要）**

```c
unsigned char depth      = png_get_bit_depth(png_ptr, info_ptr);
unsigned char color_type = png_get_color_type(png_ptr, info_ptr);

if (16 == depth)
    png_set_strip_16(png_ptr);        // 16 位 → 8 位
if (8 > depth)
    png_set_expand(png_ptr);          // 位深 < 8 扩展为 24-bit RGB
if (PNG_COLOR_TYPE_GRAY_ALPHA == color_type)
    png_set_gray_to_rgb(png_ptr);     // 灰度图 → RGB
```

libpng 原生支持不了的格式（YUV565、RGB565、YUYV 等），可注册**自定义转换函数**：
- `png_set_read_user_transform_fn()` 注册转换函数；
- `png_set_user_transform_info()` 告知输出颜色深度、通道数等信息。

**d) 更新信息**

设置后信息会变，需更新：

```c
png_read_update_info(png_ptr, info_ptr);
```

**e) 读取解码**

一次性整图解码：

```c
png_read_image(png_ptr, row_pointers);
```

`row_pointers` 是 `unsigned char **`（`png_bytepp`），一个**指针数组**，每个元素指向一行：

```c
png_bytep row_pointers[height] = {0};
size_t rowbytes = png_get_rowbytes(png_ptr, info_ptr);  // 每行字节数
int row;

for (row = 0; row < height; row++)
    row_pointers[row] = png_malloc(png_ptr, rowbytes);

png_read_image(png_ptr, row_pointers);
```

> `png_malloc()` 就是 libpng 版的 `malloc`。

也可以**逐行**读（省内存）：

```c
size_t rowbytes = png_get_rowbytes(png_ptr, info_ptr);
png_bytep row_buf = png_malloc(png_ptr, rowbytes);
int row;

for (row = 0; row < height; row++) {
    png_read_rows(png_ptr, &row_buf, NULL, 1);  // 每次 1 行，自动跳到下一行
    /* 处理这一行：如刷入 LCD 显存 */
    do_something();
}
```

**f) 结束解码**

```c
png_read_end(png_ptr, info_ptr);
```

### 21.5.7 获取解码后的数据

- **low-level**：缓冲区是自己分配的，直接从 `row_pointers` / `row_buf` 取。
- **high-level**（`png_read_png` 内部已分配缓冲区）：用 `png_get_rows()` 拿到"指向每行缓冲区的指针数组"：

```c
png_bytepp row_pointers = NULL;
row_pointers = png_get_rows(png_ptr, info_ptr);
```

销毁 `png_struct` 时，这块内部缓冲区会一并释放。

### 21.5.8 结束、销毁对象

```c
void png_destroy_read_struct(png_structpp png_ptr_ptr,
                             png_infopp info_ptr_ptr,
                             png_infopp end_info_ptr_ptr);
```

```c
png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
```

---

## 21.6 libpng 应用编程（实战）

例程路径：`21_libpng/show_png_image.c`。用 **high-level** 接口（`png_read_png`）一次性解码，得到 RGB888，再转 RGB565 刷显存。

```c
/***************************************************************
 Copyright © ALIENTEK Co., Ltd. 1998-2021. All rights reserved.
 文件名 : show_png_image.c
 作者 : 邓涛
 版本 : V1.0
 描述 : libpng 使用实战
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
#include <png.h>

static int width;                          // LCD X 分辨率
static int height;                         // LCD Y 分辨率
static unsigned short *screen_base = NULL; // 映射后的显存基地址
static unsigned long line_length;          // LCD 一行的长度（字节）
static unsigned int bpp;                   // 像素深度 bpp

static int show_png_image(const char *path)
{
    png_structp png_ptr = NULL;
    png_infop info_ptr = NULL;
    FILE *png_file = NULL;
    unsigned short *fb_line_buf = NULL;     // 存要写入 LCD 的一行
    unsigned int min_h, min_w;
    unsigned int valid_bytes;
    unsigned int image_h, image_w;
    png_bytepp row_pointers = NULL;
    int i, j, k;

    /* 打开 png 文件 */
    png_file = fopen(path, "r");
    if (NULL == png_file) {
        perror("fopen error");
        return -1;
    }

    /* 分配和初始化 png_ptr、info_ptr */
    png_ptr = png_create_read_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
    if (!png_ptr) {
        fclose(png_file);
        return -1;
    }

    info_ptr = png_create_info_struct(png_ptr);
    if (!info_ptr) {
        png_destroy_read_struct(&png_ptr, NULL, NULL);
        fclose(png_file);
        return -1;
    }

    /* 设置错误返回点 */
    if (setjmp(png_jmpbuf(png_ptr))) {
        png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
        fclose(png_file);
        return -1;
    }

    /* 指定数据源 */
    png_init_io(png_ptr, png_file);

    /* 读取 png 文件 */
    png_read_png(png_ptr, info_ptr, PNG_TRANSFORM_STRIP_ALPHA, NULL);
    image_h = png_get_image_height(png_ptr, info_ptr);
    image_w = png_get_image_width(png_ptr, info_ptr);
    printf("分辨率: %d*%d\n", image_w, image_h);

    /* 判断是不是 8 位深、RGB 颜色 */
    if ((8 != png_get_bit_depth(png_ptr, info_ptr)) &&
        (PNG_COLOR_TYPE_RGB != png_get_color_type(png_ptr, info_ptr))) {
        printf("Error: Not 8bit depth or not RGB color");
        png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
        fclose(png_file);
        return -1;
    }

    /* 判断图像和 LCD 屏哪个的分辨率更低 */
    if (image_w > width)
        min_w = width;
    else
        min_w = image_w;

    if (image_h > height)
        min_h = height;
    else
        min_h = image_h;

    valid_bytes = min_w * bpp / 8;

    /* 读取解码后的数据 */
    fb_line_buf = malloc(valid_bytes);
    row_pointers = png_get_rows(png_ptr, info_ptr);   // 获取整图数据
    unsigned int temp = min_w * 3;                    // RGB888 每像素 3 字节
    for (i = 0; i < min_h; i++) {
        /* RGB888 转为 RGB565 */
        for (j = k = 0; j < temp; j += 3, k++)
            fb_line_buf[k] = ((row_pointers[i][j]     & 0xF8) << 8) |
                             ((row_pointers[i][j + 1] & 0xFC) << 3) |
                             ((row_pointers[i][j + 2] & 0xF8) >> 3);

        memcpy(screen_base, fb_line_buf, valid_bytes); // 一行刷入显存
        screen_base += width;                          // 定位到显存下一行
    }

    /* 结束、销毁/释放内存 */
    png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
    free(fb_line_buf);
    fclose(png_file);
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
        fprintf(stderr, "usage: %s <png_file>\n", argv[0]);
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

    /* 屏幕刷白，然后显示 png */
    memset(screen_base, 0xFF, screen_size);
    show_png_image(argv[1]);

    /* 退出 */
    munmap(screen_base, screen_size);
    close(fd);
    exit(EXIT_SUCCESS);
}
```

### 代码关键点

1. **用 high-level 接口**：`png_read_png(png_ptr, info_ptr, PNG_TRANSFORM_STRIP_ALPHA, NULL)` 一次解码整图，`PNG_TRANSFORM_STRIP_ALPHA` 丢弃 alpha 通道（LCD 不透明，用不上）。
2. **`png_get_rows()` 取数据**：high-level 下缓冲区由 libpng 内部管理，必须用它拿行指针数组。
3. **RGB888 → RGB565**：
   ```c
   R5 = (r & 0xF8) << 8;   G6 = (g & 0xFC) << 3;   B5 = (b & 0xF8) >> 3;
   ```
4. **`temp = min_w * 3`**：每行只处理 `min_w` 个像素（裁剪，不缩放）。
5. **`screen_base += width`**：同第 20 章，假定 16bpp、`line_length == width*2`。
6. **`png_read_png` 的透明度处理**：`PNG_TRANSFORM_STRIP_ALPHA` 后输出 RGB888（每像素 3 字节），所以下面按 3 字节步进。

> 小提醒：原书判断 `if ((8 != bit_depth) && (PNG_COLOR_TYPE_RGB != color_type))` 用的是 `&&`，意思是"位深不是 8 **且** 颜色不是 RGB"才报错；严格来说检查 8 位深应为 `||`。这里按原书保留，理解意图即可（要求 8 位深 + RGB）。

### 编译

原书：

```bash
${CC} -o testApp testApp.c \
    -I/home/dt/tools/png/include \
    -L/home/dt/tools/png/lib -L/home/dt/tools/zlib/lib \
    -lpng -lz
```

要点：
- `-I`：**只需 libpng 的头文件目录**（zlib 头文件不用单独指定）。
- 两次 `-L`：分别给 libpng、zlib 的库目录。
- `-lpng -lz`：链接 `libpng.so`、`libz.so`。

**用独立工具链时**（实测可用）：

```bash
arm-linux-gnueabihf-gcc -o show_png_image show_png_image.c \
    -I ~/tools/png/include \
    -L ~/tools/png/lib -L ~/tools/zlib/lib \
    -lpng -lz
```

![21-22 图21.6.1 编译示例代码](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-22%20%E5%9B%BE21.6.1%20%E7%BC%96%E8%AF%91%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.png)

### 运行

可执行文件 + 一张 `.png` 拷到板子 `/home/root`（scp 带文件名）：

```bash
scp show_png_image image.png root@<板子IP>:/home/root/
```

板子上（独立库目录时设好路径，png 和 zlib 都要）：

```bash
export LD_LIBRARY_PATH=/home/root/png_lib/lib:/home/root/zlib_lib/lib:$LD_LIBRARY_PATH
./show_png_image image.png
```

执行前关闭出厂 Qt GUI。

![21-23 图21.6.2 执行测试程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-23%20%E5%9B%BE21.6.2%20%E6%89%A7%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F.png)

> 运行时会打印一些**警告信息**，这是新版 libpng 增强了检查所致，不影响使用，可忽略。

![21-24 图21.6.3 LCD 显示 png](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E4%B8%80%E7%AB%A0%20%E5%9C%A8LCD%E4%B8%8A%E6%98%BE%E7%A4%BApng%E5%9B%BE%E7%89%87/21-24%20%E5%9B%BE21.6.3%20LCD%20%E6%98%BE%E7%A4%BA%20png.jpg)

### 实测记录（ATK-IMX6U 实机）

- 板子屏幕：**1024×600，16bpp，stride=2048**。
- 库部署：`/home/root/zlib_lib`（`libz.so.1.2.10`）、`/home/root/png_lib`（`libpng16.so.16.35.0`），**独立目录，未覆盖系统 `/usr/lib`**。
- 交叉编译：SDK `arm-poky-linux-gnueabi-gcc`，zlib / libpng 均为 ARM 产物。
- 实测运行：`./show_png test.png` → 打印 `分辨率: 1024*600`，**rc=0**，LCD 正常显示。
- ⚠️ **本例程同样有第 20 章那个 `munmap` 段错误**：`screen_base` 是全局变量，写屏循环 `screen_base += width` 后 `main` 的 munmap 用偏移地址 → rc=139。**源码已改为局部 `fb_ptr` 写屏**（和第 20 章改法一致），否则直接跑必崩。
- 测试图生成：
  ```bash
  convert -size 1024x600 gradient:red-blue -depth 8 -type TrueColor test.png
  ```

### 与第 20 章对照

| 项目 | 第 20 章 jpeg | 第 21 章 png |
|------|---------------|--------------|
| 解码库 | libjpeg | libpng（依赖 zlib） |
| 解码接口 | 8 步流程，逐行 `jpeg_read_scanlines` | high-level `png_read_png` 一次搞定 |
| 解码输出 | 默认 RGB888（原书叫 BGR888） | `STRIP_ALPHA` 后 RGB888 |
| 转 RGB565 | 逐行转换 | 逐行转换 |
| 显示 | framebuffer 刷显存 | 完全相同 |
| 错误处理 | `jpeg_std_error` 默认 `exit()` | `setjmp/longjmp` 错误返回点 |

**记住一条主线**：不管 jpg 还是 png，最终都是"解码 → RGB888 → RGB565 → 逐行 memcpy 到 framebuffer"，区别只在解码库和 API。

---

## 附：本章移植顺序速查

```bash
# 1. zlib
tar -xzf zlib-1.2.10.tar.gz
cd zlib-1.2.10
./configure --prefix=$HOME/tools/zlib --host=arm-linux-gnueabihf
make && make install

# 2. libpng（必须先告诉它 zlib 在哪）
tar -xzf libpng-1.6.35.tar.gz
cd libpng-1.6.35
export LDFLAGS="$LDFLAGS -L$HOME/tools/zlib/lib"
export CFLAGS="$CFLAGS -I$HOME/tools/zlib/include"
export CPPFLAGS="$CPPFLAGS -I$HOME/tools/zlib/include"
./configure --prefix=$HOME/tools/png --host=arm-linux-gnueabihf
make && make install

# 3. 编译例程
arm-linux-gnueabihf-gcc -o show_png_image show_png_image.c \
    -I$HOME/tools/png/include -L$HOME/tools/png/lib -L$HOME/tools/zlib/lib -lpng -lz
```
