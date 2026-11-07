# 第四章 标准 I/O 库

> 前两章用的 `open`/`read`/`write` 是**系统调用**，跨平台就废、每次调用都可能触发磁盘操作。标准 I/O 库（`fopen`/`fread`/`fwrite`/`fclose`…）是 C 标准库在系统调用之上封的一层：**接口统一可移植**，且**在用户空间自带 stdio 缓冲区**，把多次小 I/O 合并成一次系统调用，效率高得多。代价是——缓冲区带来一堆“明明写了却没落盘”“printf 没输出”“printf 和 write 顺序错乱”的怪现象，这一章有一半篇幅就是在讲这个。
>
> 本章主题：标准 I/O 与文件 I/O 的区别、FILE 指针、stdin/stdout/stderr、`fopen`/`fclose`、`fread`/`fwrite`、`fseek`/`ftell`、`feof`/`ferror`/`clearerr`、格式化输出与输入、内核缓冲与 stdio 缓冲、直接 I/O、`fileno`/`fdopen`。

一句话主线：**fd 面向内核，FILE\* 面向用户；中间隔着 stdio 缓冲区，再下面还有内核缓冲区**。任何“数据没到该到的地方”的问题，先问“卡在哪一层缓冲”。

---

## 4.1 标准 I/O 库简介

标准 I/O 库是标准 C 库中用于文件 I/O 的一系列库函数的集合，定义在 `<stdio.h>`。它**构建于文件 I/O 系统调用之上**：`fopen()` 内部用 `open()`，`fread()` 内部用 `read()`，`fwrite()` 内部用 `write()`。

既然底层就是系统调用，为什么还要多这一层？

| 维度 | 标准 I/O | 文件 I/O |
|---|---|---|
| 本质 | **标准 C 库函数** | **Linux 系统调用** |
| 关系 | 由文件 I/O 封装而来，内部调用文件 I/O | 直接向内核请求 |
| 可移植性 | **好**：各系统接口几乎一致 | 差：不同系统定义、功能、参数、返回值都可能有差异 |
| 性能 | **高**：用户空间有 stdio 缓冲区 | 低：用户空间不带缓存 |
| 操作对象 | `FILE *` | `int fd` |

⚫ 说“文件 I/O 不带缓存”是指**用户空间**不带缓存；内核里仍有内核缓冲区（见 4.9）。

---

## 4.2 FILE 指针

文件 I/O 围绕**文件描述符 fd** 转；标准 I/O 围绕 **FILE 指针（`FILE *`）** 转。打开/创建文件时返回一个指向 FILE 类型对象的指针，该指针与文件关联，后续标准 I/O 操作都用它。

**FILE 指针的作用相当于文件描述符**，只是使用场合不同：FILE 指针给标准 I/O 库函数用，fd 给系统调用用。

FILE 是一个**结构体类型**，包含标准 I/O 库管理文件所需的全部信息：

- 用于实际 I/O 的**文件描述符**；
- 指向**文件缓冲区**的指针；
- **缓冲区的长度**；
- 当前**缓冲区中的字节数**；
- **出错标志**。

FILE 定义在 `<stdio.h>` 中（glibc 里实体叫 `struct _IO_FILE`，`FILE` 是它的 typedef）。

---

## 4.3 标准输入、标准输出和标准错误

| 概念 | 设备 | fd | 标准 I/O 的 FILE 指针 |
|---|---|---|---|
| 标准输入 stdin | 键盘 | 0 | `stdin` |
| 标准输出 stdout | 显示器/终端 | 1 | `stdout` |
| 标准错误 stderr | 显示器/终端 | 2 | `stderr` |

进程通过标准输入读用户输入，正常输出（如 `printf`）走标准输出，错误信息走标准错误。stdout 与 stderr 都对应终端屏幕，stdin 对应键盘。

每个进程启动后**默认已打开**这三个文件，得到 fd 0/1/2。`<unistd.h>` 中的宏：

```c
#define STDIN_FILENO    0    /* Standard input. */
#define STDOUT_FILENO   1    /* Standard output. */
#define STDERR_FILENO   2    /* Standard error output. */
```

0/1/2 是 fd，只能用于文件 I/O；标准 I/O 里对应的对象定义在 `<stdio.h>`：

```c
extern struct _IO_FILE *stdin;
extern struct _IO_FILE *stdout;
extern struct _IO_FILE *stderr;
```

⚫ `struct _IO_FILE` 就是 `FILE`（typedef 重命名）。
⚫ `stdin`/`stdout`/`stderr` 是**宏**，本质是 FILE 指针。

---

## 4.4 打开文件 fopen()

```c
#include <stdio.h>

FILE *fopen(const char *path, const char *mode);
```

| 参数 | 含义 |
|---|---|
| `path` | 文件路径，绝对或相对 |
| `mode` | 读写权限，**字符串** |

**返回值**：成功返回指向 FILE 对象的指针；失败返回 **NULL** 并设置 `errno`。

`mode` 取值（对应 `open()` 的 flags）：

| mode | 说明 | 对应 open flags |
|---|---|---|
| `"r"` | 只读打开 | `O_RDONLY` |
| `"r+"` | 可读可写打开 | `O_RDWR` |
| `"w"` | 只写打开；文件存在则**截断为 0**，不存在则创建 | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `"w+"` | 可读可写打开；存在则截断为 0，不存在则创建 | `O_RDWR \| O_CREAT \| O_TRUNC` |
| `"a"` | 只写**追加**（写末尾）；不存在则创建 | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `"a+"` | 可读可写**追加**；不存在则创建 | `O_RDWR \| O_CREAT \| O_APPEND` |

⚫ `"w"`/`"w+"` 会**静默清空**已有文件，这是最常见的误删数据原因。
⚫ `"a"`/`"a+"` 的追加靠 `O_APPEND`，因此是原子追加（第 3 章结论）。

### 4.4.1 新建文件的权限

`fopen()` 只有两个参数，**无法手动指定新文件权限**。用 `"w"`、`"w+"`、`"a"`、`"a+"` 创建新文件时，权限取默认值：

```c
S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH   /* 即 0666 */
```

⚫ 最终落地权限还要再被进程 **umask** 削减（见 5.5）。
⚫ 需要精确控制权限时，用 `open()` 而不是 `fopen()`。

### 4.4.2 fclose 关闭文件

```c
#include <stdio.h>

int fclose(FILE *stream);
```

| 参数 | 含义 |
|---|---|
| `stream` | FILE 指针 |

**返回值**：成功 0；失败返回 **EOF（即 -1）** 并设置 `errno`。

⚫ `fclose()` 关闭时会**自动刷新该文件的 stdio 缓冲区**（见 4.9）。

---

## 4.5 读文件和写文件

```c
#include <stdio.h>

size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

| 参数 | 含义 |
|---|---|
| `ptr` | `fread`：读到该缓冲区；`fwrite`：从该缓冲区写出 |
| `size` | **每个数据项的字节大小** |
| `nmemb` | **数据项个数** |
| `stream` | FILE 指针 |

总数据量 = `nmemb * size` 字节。

**返回值**：

- `fread`：成功返回**读到的数据项个数**（不是字节数，除非 `size == 1`）；出错或到达文件末尾时返回值小于 `nmemb`。**`fread` 无法区分“出错”和“文件结束”**，必须用 `feof()` / `ferror()` 判断（4.7）。
- `fwrite`：成功返回**写入的数据项个数**；出错时返回值小于 `nmemb`（或为 0）。

写法对比：

```c
fwrite(buf, sizeof(struct mystr), 1, file);   /* 1 项，每项 sizeof(struct mystr) */
fwrite(buf, 1, sizeof(struct mystr), file);   /* sizeof 项，每项 1 字节 */
```

⚫ 两种都能写同样的数据；**返回值语义不同**（前者成功为 1，后者成功为 sizeof）。判断“写全了没有”要按自己的 `size` 写。
⚫ 与 `read`/`write` 的核心差别：后者只有一个 `size` 参数直接给字节数，前者是 `nmemb * size`。

### 4.5.1 写文件示例

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    char buf[] = "Hello World!\n";
    FILE *fp = NULL;

    if (NULL == (fp = fopen("./test_file", "w"))) {
        perror("fopen error");
        exit(-1);
    }
    printf("文件打开成功!\n");

    if (sizeof(buf) > fwrite(buf, 1, sizeof(buf), fp)) {
        printf("fwrite error\n");
        fclose(fp);
        exit(-1);
    }
    printf("数据写入成功!\n");

    fclose(fp);
    exit(0);
}
```

![图4.5.1 测试结果 示例代码 4.5.2 演示了](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-1%20%E5%9B%BE4.5.1%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81%204.5.2%20%E6%BC%94%E7%A4%BA%E4%BA%86.png)

### 4.5.2 读文件示例

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    char buf[50] = {0};
    FILE *fp = NULL;
    int size;

    if (NULL == (fp = fopen("./test_file", "r"))) {
        perror("fopen error");
        exit(-1);
    }
    printf("文件打开成功!\n");

    if (12 > (size = fread(buf, 1, 12, fp))) {
        if (ferror(fp)) {          /* 是错误 */
            printf("fread error\n");
            fclose(fp);
            exit(-1);
        }
        /* 未出错 → 说明到达文件末尾 */
    }

    printf("成功读取%d 个字节数据: %s\n", size, buf);

    fclose(fp);
    exit(0);
}
```

![图4.5.2 测试结果](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-2%20%E5%9B%BE4.5.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

⚫ 这里判断用 `12 > size` 而不是 `12 != size`：短读时才是“出错或 EOF”，超读不可能发生。
⚫ **必须先判 `ferror` 再认定 EOF**，顺序反了会把错误当正常结束。

---

## 4.6 fseek 定位

```c
#include <stdio.h>

int fseek(FILE *stream, long offset, int whence);
```

| 参数 | 含义 |
|---|---|
| `stream` | FILE 指针 |
| `offset` | 偏移量，字节 |
| `whence` | 基准：`SEEK_SET` / `SEEK_CUR` / `SEEK_END`，同 `lseek` |

**返回值**：成功 **0**；失败 **-1** 并设置 `errno`。

⚫ **这是与 `lseek` 最大的差别**：`lseek` 成功返回“新偏移量”，`fseek` 成功只返回 0。别把返回值当偏移量用。

常用写法：

```c
fseek(file, 0, SEEK_SET);     /* 移到文件头 */
fseek(file, 0, SEEK_END);     /* 移到文件末尾 */
fseek(file, 100, SEEK_SET);   /* 移到距文件头 100 字节处 */
```

`fread`/`fwrite` 后偏移量会自动递增，需要回头读写就用 `fseek` 手动设。

### 4.6.1 使用示例

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    FILE *fp = NULL;
    char rd_buf[100] = {0};
    char wr_buf[] = "正点原子 http://www.openedv.com/forum.php\n";
    int ret;

    if (NULL == (fp = fopen("./test_file", "w+"))) {
        perror("fopen error");
        exit(-1);
    }
    printf("文件打开成功!\n");

    if (sizeof(wr_buf) > fwrite(wr_buf, 1, sizeof(wr_buf), fp)) {
        printf("fwrite error\n");
        fclose(fp);
        exit(-1);
    }
    printf("数据写入成功!\n");

    /* 写完偏移已在末尾，必须回到文件头才能读到 */
    if (0 > fseek(fp, 0, SEEK_SET)) {
        perror("fseek error");
        fclose(fp);
        exit(-1);
    }

    if (sizeof(wr_buf) > (ret = fread(rd_buf, 1, sizeof(wr_buf), fp))) {
        printf("fread error\n");
        fclose(fp);
        exit(-1);
    }

    printf("成功读取%d 个字节数据: %s\n", ret, rd_buf);

    fclose(fp);
    exit(0);
}
```

![图4.6.1 测试结果](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-3%20%E5%9B%BE4.6.1%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

### 4.6.2 ftell

```c
#include <stdio.h>

long ftell(FILE *stream);
```

**返回值**：成功返回**当前读写位置偏移量**；失败返回 -1 并设置 `errno`。

配合 `fseek(fp, 0, SEEK_END)` 可求文件大小：

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    FILE *fp = NULL;
    int ret;

    if (NULL == (fp = fopen("./testApp.c", "r"))) {
        perror("fopen error");
        exit(-1);
    }
    printf("文件打开成功!\n");

    if (0 > fseek(fp, 0, SEEK_END)) {
        perror("fseek error");
        fclose(fp);
        exit(-1);
    }

    if (0 > (ret = ftell(fp))) {
        perror("ftell error");
        fclose(fp);
        exit(-1);
    }

    printf("文件大小: %d 个字节\n", ret);

    fclose(fp);
    exit(0);
}
```

![图4.6.2 测试结果 从](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-4%20%E5%9B%BE4.6.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E4%BB%8E.png)

结果与 `ls` 看到的文件大小一致。

---

## 4.7 检查或复位状态

`fread` 返回值小于 `nmemb` 时，可能是**出错**也可能是**到达文件末尾**，`fread` 自己不区分，要靠下面三个函数。

### 4.7.1 feof

```c
#include <stdio.h>

int feof(FILE *stream);
```

**返回值**：end-of-file 标志被设置 → **非零**；未设置 → **0**。

读写位置移动到文件末尾时，该标志被设置。

```c
if (feof(file)) { /* 到达文件末尾 */ }
else             { /* 未到达 */ }
```

### 4.7.2 ferror

```c
#include <stdio.h>

int ferror(FILE *stream);
```

**返回值**：错误标志被设置 → **非零**；未设置 → **0**。

文件 I/O 操作发生错误时错误标志被设置。

```c
if (ferror(file)) { /* 发生错误 */ }
else              { /* 未发生错误 */ }
```

### 4.7.3 clearerr

```c
#include <stdio.h>

void clearerr(FILE *stream);
```

**返回值**：无（`void`），调用总是成功。用于**同时清除** end-of-file 标志和错误标志。

⚫ 标志是**粘滞**的：一旦置位就不会自己复位，下次 `feof`/`ferror` 还会读到上次的结果，因此校验完要 `clearerr()`。
⚫ 例外：调用 `fseek()` 成功时也会**自动清除 end-of-file 标志**。

### 4.7.4 使用示例

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    FILE *fp = NULL;
    char buf[20] = {0};

    if (NULL == (fp = fopen("./testApp.c", "r"))) {
        perror("fopen error");
        exit(-1);
    }
    printf("文件打开成功!\n");

    /* 故意移到末尾，制造 EOF 情形 */
    if (0 > fseek(fp, 0, SEEK_END)) {
        perror("fseek error");
        fclose(fp);
        exit(-1);
    }

    if (10 > fread(buf, 1, 10, fp)) {
        if (feof(fp))
            printf("end-of-file 标志被设置,已到文件末尾!\n");
        clearerr(fp);       /* 清除标志 */
    }

    fclose(fp);
    exit(0);
}
```

⚫ 上例中 `fseek` 会先清掉 EOF 标志，所以那次读取之后标志才是“本次”置上的。

---

## 4.8 格式化 I/O

### 4.8.1 格式化输出

```c
#include <stdio.h>

int printf(const char *format, ...);
int fprintf(FILE *stream, const char *format, ...);
int dprintf(int fd, const char *format, ...);
int sprintf(char *buf, const char *format, ...);
int snprintf(char *buf, size_t size, const char *format, ...);
```

| 函数 | 输出去向 |
|---|---|
| `printf` | 标准输出 |
| `fprintf` | 由 **FILE 指针** 指定的文件 |
| `dprintf` | 由**文件描述符 fd** 指定的文件 |
| `sprintf` | 用户缓冲区 `buf`（**无长度限制，危险**） |
| `snprintf` | 用户缓冲区 `buf`，最多 `size` 字节（安全） |

**返回值**：成功返回输出/写入的**字符数**；失败返回**负值**。

`fprintf` 与 `dprintf` 的区别只有一点：一个用 `FILE*`，一个用 `fd`。

```c
printf("Hello World!\n");
printf("%d\n", 5);

fprintf(stderr, "Hello World!\n");
fprintf(stderr, "%d\n", 5);

dprintf(STDERR_FILENO, "Hello World!\n");
dprintf(STDERR_FILENO, "%d\n", 5);

char buf[20] = {0};
sprintf(buf, "%d", 100);          /* 数字转字符串 "100" */
```

⚫ `sprintf` 会在末尾自动加 `'\0'`，但**不检查 buf 大小**，溢出会造成程序不稳定甚至安全隐患。**有格式化需求优先用 `snprintf`**。
⚫ `snprintf` 用 `size` 显式限长，超出部分**被丢弃**；缓冲区足够大时返回写入字符数；同样自动补 `'\0'`；出错返回负值。`size` 包含 `'\0'` 的位置，所以最多写入 `size-1` 个字符。

### 4.8.2 格式控制字符串 format

格式串由两部分组成：**普通字符**（非 `%`，原样输出）和**转换说明**。每个转换说明对应后续一个参数，按顺序一一对应。

```c
printf("转换说明1 转换说明2 转换说明3", arg1, arg2, arg3);
```

转换说明格式（`[]` 内为可选）：

```
%[flags][width][.precision][length]type
```

⚫ **只有 `%` 和 `type` 是必须的**，其余都可选。

#### (1) type 类型

| 字符 | 数据类型 | 含义 | 示例 |
|---|---|---|---|
| `d`/`i` | `int` | 有符号十进制整数（`i` 是老式写法） | `printf("%d\n",123)` → 123 |
| `o` | `unsigned int` | 无符号八进制（默认无前缀 0，加 `#` 输出 0） | `printf("%o\n",123)` → 173 |
| `u` | `unsigned int` | 无符号十进制 | `printf("%u\n",123)` → 123 |
| `x`/`X` | `unsigned int` | 无符号十六进制，`x` 用 abcdef、`X` 用 ABCDEF；默认无前缀，加 `#` 输出 0x/0X | `%x`→7b，`%X`→7B |
| `f`/`F` | `double` | 浮点数，未指定精度时**默认保留 6 位小数** | `printf("%f\n",520.1314)` → 520.131400 |
| `e`/`E` | `double` | 科学计数法，区别在 e 的大小写 | → 5.201314e+02 |
| `g`/`G` | `double` | 按数值长度选最短方式输出（`%f`/`%e` 二选一） | `%g` 0.000000123 → 1.23e-07 |
| `s` | `char *` | 字符串，输出至 `'\0'` | `printf("%s\n","Hello World")` |
| `p` | `void *` | 十六进制指针 | `printf("%p\n",ptr)` → 0x400624 |
| `c` | `char` | 字符，数字按 ASCII 转换 | `printf("%c\n",64)` → A |

#### (2) flags

| 字符 | 名称 | 作用 |
|---|---|---|
| `#` | 井号 | `o` 加前缀 0；`x`/`X` 加前缀 0x/0X；浮点（a/A/e/E/f/F/g/G）时始终包含小数点，`g`/`G` 保留尾部 0 |
| `0` | 数字零 | `type` 不是 `c`/`s` 时，在前面补 0 直到占满最小宽度（如宽度 5 输出 100 → 00100）；不指定则补空格 |
| `-` | 减号 | 默认右对齐（左侧补空格或 0）；用它变**左对齐**，右侧补空格。同时指定 `0` 和 `-` 时 `-` 覆盖 `0` |
| `' '` | 空格 | 正数前加空格，负数前加 `-` |
| `+` | 加号 | 所有数字都带符号（正数 `+`、负数 `-`）；同时指定 `+` 和空格时 `+` 覆盖空格 |

#### (3) width

输出**最小宽度**（十进制位数）。实际位数大于它则按实际输出；小于它则按 flags 补 0 或补空格。

| 取值 | 说明 | 示例 |
|---|---|---|
| 数字 | 十进制正数 | `printf("%06d",1000)` → 001000 |
| `*` | 不写死宽度，由**参数列表**给出 | `printf("%0*d",6,1000)` → 001000 |

#### (4) precision 精度

以点号 `.` 开头，后跟十进制正数：

| 类型 | 含义 |
|---|---|
| 整型（`d`/`i`/`o`/`u`/`x`/`X`） | **最小数字个数**，不足补前导零，超过不截断。注意是“数字位数”，与 width 的“整个字符串宽度”不同；此时**忽略 flags 的 `0`** |
| 浮点（`a`/`A`/`e`/`E`/`f`/`F`） | **小数点后位数**，默认 6 位，不足补 0，超过截断 |
| `g`/`G` | **最大有效位数** |
| 字符串（`s`） | **最大输出字符数**，超出截断 |
| `*` | 由参数列表给出精度 |

```c
printf("%8.5d\n", 100);        /* 数字补到 5 位 → 00100，再补 3 个空格到总宽 8 */
printf("%.8f\n", 520.1314);    /* 520.13140000 */
printf("%.5s\n", "hello world");   /* hello */
printf("%.*s\n", 5, "hello world"); /* hello */
```

#### (5) length 长度修饰符

`type` 只能表达 `int`/`unsigned int`/`double` 这几种，其它长度靠 length 修饰：

| length | `d`/`i` | `u`/`o`/`x`/`X` | `f`/`F`/`e`/`E`/`g`/`G` | `c` | `s` | `p` |
|---|---|---|---|---|---|---|
| 无 | `int` | `unsigned int` | `double` | `char` | `char *` | `void *` |
| `hh` | `signed char` | `unsigned char` | — | — | — | — |
| `h` | `short int` | `unsigned short int` | — | — | — | — |
| `l` | `long int` | `unsigned long int` | — | `wint_t` | `wchar_t *` | — |
| `ll` | `long long int` | `unsigned long long int` | — | — | — | — |
| `L` | — | — | `long double` | — | — | — |
| `j` | `intmax_t` | `uintmax_t` | — | — | — | — |
| `z` | `size_t` | `ssize_t` | — | — | — | — |
| `t` | `ptrdiff_t` | `ptrdiff_t` | — | — | — | — |

```c
printf("%hd\n", 12345);    /* short int */
printf("%ld\n", 12345);    /* long int */
printf("%lld\n", 12345);   /* long long int */
```

⚫ 格式说明与参数类型**必须匹配**，否则编译报错或警告；不匹配但能编译时，运行结果是未定义的。
⚫ 这些不用背，用时查表。

#### (6) 综合示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main(void)
{
    char buf[50] = {0};

    printf("%d (%s) %d (%s)\n", 520, "我爱你", 1314, "一生一世");
    fprintf(stdout, "%d (%s) %d (%s)\n", 520, "我爱你", 1314, "一生一世");
    dprintf(STDOUT_FILENO, "%d (%s) %d (%s)\n", 520, "我爱你", 1314, "一生一世");

    sprintf(buf, "%d (%s) %d (%s)\n", 520, "我爱你", 1314, "一生一世");
    printf("%s", buf);

    memset(buf, 0x00, sizeof(buf));
    snprintf(buf, sizeof(buf), "%d (%s) %d (%s)\n", 520, "我爱你", 1314, "一生一世");
    printf("%s", buf);

    exit(0);
}
```

![图4.8.1 运行结果 关于格式化输出](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-5%20%E5%9B%BE4.8.1%20%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C%20%E5%85%B3%E4%BA%8E%E6%A0%BC%E5%BC%8F%E5%8C%96%E8%BE%93%E5%87%BA.png)

### 4.8.3 格式化输入

```c
#include <stdio.h>

int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
int sscanf(const char *str, const char *format, ...);
```

| 函数 | 输入来源 |
|---|---|
| `scanf` | 标准输入（键盘） |
| `fscanf` | `FILE *` 指定的文件（传 `stdin` 就等价于 `scanf`） |
| `sscanf` | 字符串 `str` |

**返回值**：成功返回**成功匹配并赋值的输入项数量**；若较早匹配失败，该数目可能小于提供数目、甚至为 0；发生错误返回负值。

工作方式：从 format 最左端开始，每遇到一个转换说明就与下一个输入数据“匹配”，匹配成功继续、失败则停止后续处理；每个转换说明按描述转换后存入对应**地址**（必须传指针）。

```c
int a, b, c;
scanf("%d %d %d", &a, &b, &c);      /* 调用后终端阻塞，回车才继续 */

char *str = "5454 hello";
char buf[10];
int n;
sscanf(str, "%d %s", &n, buf);
```

⚫ **必须传地址**（`&a`、`buf`），传值会崩溃。
⚫ 转换说明与参数必须一一对应，数量不匹配同样是未定义行为。
⚫ `scanf` 会阻塞等待输入，这是标准输入行缓冲导致的（见 4.9）。

#### (1) format 结构

转换说明以 `%` 或 `%n$` 开头（后者用得少），一般格式：

```
%[*][width][length]type
%[m][width][length]type
```

| 修饰 | 含义 |
|---|---|
| `*` | 按说明读取但**丢弃**输入，不存储，因此**不需要提供对应指针参数** |
| `m` | 只能与 `%s`、`%c`、`%[` 一起用；由函数**内部自动分配**足够大的缓冲区，并把地址通过对应指针参数返回（该参数应是 `char *` 的指针）。用完必须 `free()` |
| `width` | **最大字符宽度** |
| `length` | 长度修饰符，同输出 |
| `type` | 输入数据类型，**必须** |

`m` 用法：

```c
char *buf;
scanf("%ms", &buf);
/* ... 使用 buf ... */
free(buf);
```

#### (2) type 类型

| 字符 | 数据类型 | 含义 |
|---|---|---|
| `d` | `int` | 匹配有符号十进制整数 |
| `i` | `int` | 匹配有符号整数，**自动识别进制**：`0x`/`0X` 开头为 16 进制，`0` 开头为 8 进制，否则 10 进制 |
| `o` | `unsigned int` | 无符号八进制 |
| `u` | `unsigned int` | 无符号十进制 |
| `x` | `unsigned int` | 无符号十六进制，数字以 `0x` 开头 |
| `X` | `unsigned int` | 同上，以 `0X` 开头 |
| `f` | `float` | 带符号浮点数 |
| `e` | `float` | 等效于 `f` |
| `E` | `float` | 等效于 `f` |
| `g` | `float` | 等效于 `f` |
| `a` | `float` | 等效于 `f` |
| `s` | `char *` | 匹配字符串，**不匹配空白字符**（空格/制表符/换行符作分隔符），即匹配一串非空白字符；**自动补 `'\0'`**，缓冲区必须足够大 |
| `c` | `char` | 匹配一个字符 |
| `p` | `void *` | 匹配一个指针值 |
| `[` | `char *` | 匹配一个**字符集合**，以字符串形式存储 |

`%[` 的规则（易错，单独记）：

- `%[a-z]` 匹配 a 到 z 之间所有字符（含首尾）；
- `%[0-9]` 匹配 0~9 所有数字字符；
- `-` 是连字符，放在两个字符之间表示范围，**本身不参与匹配**；要匹配 `-` 本身，把它放在中括号**旁边**，如 `%[-a-z]`；
- 要**排除**这些字符，用 `^` 放最前面，如 `%[^a-z-]` 表示匹配除 a~z 和 `-` 之外的字符。

```c
char buf[100];
scanf("%[a-z0-9]", buf);    /* 输入 123abc，匹配字母与数字 */
```

#### (3) width 最大字符宽度

十进制整数，指定最多读多少字符；达到该值或遇到不匹配字符（先发生者为准）即停止。

⚫ 大多数 type 会**丢弃前导空白字符**，且这些被丢的字符**不计入**最大宽度。
⚫ 对字符串转换，自动补的 `'\0'` **不计入**最大宽度。

```c
scanf("%4s", buf);    /* 输入 abcdefg → 只有 abcd 存入 buf */
```

#### (4) length 长度修饰符

| length | `d`/`i` | `u`/`o`/`x`/`X` | `e`/`f`/`g` | `c` | `s` |
|---|---|---|---|---|---|
| 无 | `int` | `unsigned int` | `float` | `char` | `char *` |
| `h` | `short int` | `unsigned short int` | — | — | — |
| `hh` | `signed char` | `unsigned char` | — | — | — |
| `j` | `intmax_t` | `uintmax_t` | — | — | — |
| `l` | `long int` | `unsigned long int` | `double` | `wchar_t` | `wchar_t *` |
| `L` | `long long int` | `unsigned long long int` | `long double` | — | — |

```c
scanf("%hd",  &var);   /* short int */
scanf("%hhd", &var);   /* signed char */
scanf("%ld",  &var);   /* long int */
scanf("%f",   &var);   /* float */
scanf("%lf",  &var);   /* double */
scanf("%Lf",  &var);   /* long double */
```

⚫ **浮点的坑**：`scanf` 里 `%f` 对应 `float`、`%lf` 对应 `double`；而 `printf` 里 `%f` 本身就对应 `double`。两边规则相反，写错就会读出垃圾值。

#### (5) 示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void)
{
    int a;
    float b;
    char *str;

    printf("请输入一个整数:\n");
    scanf("%d", &a);
    printf("你输入的整数为: %d\n", a);

    printf("请输入一个浮点数:\n");
    scanf("%f", &b);
    printf("你输入的浮点数为: %f\n", b);

    printf("请输入一个字符串:\n");
    scanf("%ms", &str);        /* m：内部自动分配缓冲区 */
    printf("你输入的字符串为: %s\n", str);

    free(str);                 /* 必须释放 */
    exit(0);
}
```

![图4.8.2 测试结果](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-6%20%E5%9B%BE4.8.2%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

⚫ 连续用 `scanf("%d")` 后接 `scanf("%c")` 会读到上次残留的换行符——这是“丢弃前导空白”规则的反面：`%c` **不**跳过空白。
⚫ 格式化 I/O 的细节问题大多归根到底还是缓冲问题，见 4.9。

---

## 4.9 I/O 缓冲

出于速度和效率，系统调用 I/O 与标准 I/O 在操作磁盘时都会缓冲数据。这一节讲**两层缓冲**和绕开它们的手段。

### 4.9.1 文件 I/O 的内核缓冲

`read()`/`write()` **不直接访问磁盘**，只在**用户空间缓冲区**与**内核缓冲区（kernel buffer cache）**之间复制数据：

```c
write(fd, "Hello", 5);   /* 仅仅把 5 字节拷到内核缓冲区就返回 */
```

拷贝完成函数就返回，之后某个时刻内核才把数据写入磁盘 —— 也就是说 **`write()` 与磁盘操作不同步**。若期间别的进程 `read()` 这个文件，内核会直接从缓冲区给它数据。

读同理：内核先把数据从磁盘读进内核缓冲区，`read()` 从缓冲区取；缓冲区读完，内核再读下一段。

这样设计的好处：

- `read`/`write` 不必等待缓慢的磁盘操作，调用更快；
- 减少磁盘操作次数 —— 两个线程分别 `write` "abcd"、"1234"，数据都进内核缓冲区，稍后**一次**磁盘写就能完成；没有缓冲区则每次 `write` 都要一次磁盘操作。

⚫ 数据**何时真正落盘不确定**，由内核按存储算法决定。
⚫ 内核缓冲区**没有固定上限**，受物理内存总量限制：内存越大，可用缓冲越大。

### 4.9.2 刷新内核缓冲区

有些程序必须在某步之前确保数据已经真正落盘（如数据库日志进程）。日常场景：往 U 盘拷文件，拔盘前要 `sync`，否则可能损坏文件——`sync` 就是把内核缓冲区刷到 U 盘硬件。

#### (1) fsync

```c
#include <unistd.h>

int fsync(int fd);
```

把 `fd` 所指文件的**内容数据 + 元数据**写入磁盘；**只有磁盘写操作完成后才返回**。成功 0，失败 -1 并设置 `errno`。

⚫ **元数据**不是文件内容，而是记录文件属性的数据：大小、时间戳、权限等（第 3 章 inode 里那些）。

```c
#define BUF_SIZE   4096
#define READ_FILE  "./rfile"
#define WRITE_FILE "./wfile"

static char buf[BUF_SIZE];

int main(void)
{
    int rfd, wfd;
    size_t size;

    rfd = open(READ_FILE, O_RDONLY);
    if (0 > rfd) { perror("open error"); exit(-1); }

    wfd = open(WRITE_FILE, O_WRONLY | O_CREAT | O_TRUNC, 0664);
    if (0 > wfd) { perror("open error"); exit(-1); }

    /* 边读边写，read 返回 0 表示读完 */
    while (0 < (size = read(rfd, buf, BUF_SIZE)))
        write(wfd, buf, size);

    /* 确保目标文件数据真正落盘 */
    fsync(wfd);

    close(rfd);
    close(wfd);
    exit(0);
}
```

#### (2) fdatasync

```c
#include <unistd.h>

int fdatasync(int fd);
```

与 `fsync` 类似，区别：**只写文件内容数据，不包括元数据**；同样等磁盘写完才返回。

#### (3) sync

```c
#include <unistd.h>

void sync(void);
```

把**所有**文件 I/O 内核缓冲区中的内容数据与元数据全部更新到磁盘。**无参数、无返回值**。

⚫ Linux 实现中，`sync()` 要等所有数据写完才返回；某些其它系统只是调度一下 I/O 就返回。

### 4.9.3 控制内核缓冲的 open 标志

| 标志 | 效果 | 等价于 |
|---|---|---|
| `O_DSYNC` | 每次 `write` 自动把**数据**刷到磁盘 | 每次 `write` 后调 `fdatasync()` |
| `O_SYNC` | 每次 `write` 自动把**数据 + 元数据**刷到磁盘 | 每次 `write` 后调 `fsync()` |

```c
fd = open(filepath, O_WRONLY | O_DSYNC);
fd = open(filepath, O_WRONLY | O_SYNC);
```

⚫ 频繁 `fsync`/`fdatasync`/`sync` 或用 `O_DSYNC`/`O_SYNC`，**对性能影响极大**，绝大多数程序不需要。

### 4.9.4 直接 I/O：绕过内核缓冲

Linux 2.4 起允许应用绕过内核缓冲区，从用户空间直接把数据传给文件/磁盘设备，称**直接 I/O（direct I/O）**或**裸 I/O（raw I/O）**。

**适用场合**：需要保证读写真的直达磁盘，如磁盘速率测试工具、数据库系统。

**代价**：失去内核的优化（顺序预读取、成簇块 I/O、多进程共享缓存），性能大幅下降。**绝大多数应用不该用**。

用法：`open` 时指定 `O_DIRECT`（Linux 2.4.10 起生效）。

```c
fd = open(filepath, O_WRONLY | O_DIRECT);
```

#### (1) 三个对齐限制

直接 I/O 直接访问磁盘，必须同时满足：

1. 存放数据的**缓冲区内存起始地址**必须是块大小的整数倍；
2. 写文件时**文件位置偏移量**必须是块大小的整数倍；
3. **写入数据大小**必须是块大小的整数倍。

任何一条不满足，`write()` 返回错误 **Invalid argument**。

这里“块大小”指磁盘设备的**物理块大小**，常见 512/1024/2048/4096 字节。查看方法：

```bash
df -h                                   # 先看根文件系统挂在哪个分区
tune2fs -l /dev/sda1 | grep "Block size"   # 再看该分区块大小
```

![图4.9.1 查看根文件系统挂载的磁盘分区 通过](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-7%20%E5%9B%BE4.9.1%20%E6%9F%A5%E7%9C%8B%E6%A0%B9%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E6%8C%82%E8%BD%BD%E7%9A%84%E7%A3%81%E7%9B%98%E5%88%86%E5%8C%BA%20%E9%80%9A%E8%BF%87.png)

![图4.9.2 磁盘块大小 从](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-8%20%E5%9B%BE4.9.2%20%E7%A3%81%E7%9B%98%E5%9D%97%E5%A4%A7%E5%B0%8F%20%E4%BB%8E.png)

例中 `/dev/sda1` 块大小为 4096 字节。

#### (2) 直接 I/O 示例

⚫ 用 `O_DIRECT` 必须先 `#define _GNU_SOURCE`，否则编译报 “O_DIRECT 未定义”。`_GNU_SOURCE` 用于开启/禁用系统调用和 glibc 的部分特性，定义它意味着打开全部特性，默认未定义。

两种定义方式：

```c
#define _GNU_SOURCE                       /* 源文件开头 */
```
```bash
gcc -D_GNU_SOURCE -o testApp testApp.c    # 编译时定义，全局生效
```

```c
/** 使用 O_DIRECT 需先定义 _GNU_SOURCE，否则提示找不到 O_DIRECT **/
#define _GNU_SOURCE

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

/** 缓冲区起始地址按 4096 字节对齐 **/
static char buf[8192] __attribute((aligned(4096)));

int main(void)
{
    int fd;
    int count;

    fd = open("./test_file", O_WRONLY | O_CREAT | O_TRUNC | O_DIRECT, 0664);
    if (0 > fd) { perror("open error"); exit(-1); }

    count = 10000;
    while (count--) {
        if (4096 != write(fd, buf, 4096)) {
            perror("write error");
            exit(-1);
        }
    }

    close(fd);
    exit(0);
}
```

三条对齐要求如何满足：

- 缓冲区：`__attribute((aligned(4096)))` 让 `buf` 起始地址按 4096 对齐（`__attribute` 是 gcc 机制，也可写 `__attribute__`，可设函数/变量/类型属性）；
- 偏移量：首次 `write` 偏移为 0，之后每次 +4096，始终是块大小整数倍；
- 数据大小：每次固定写 4096 字节。

总共写入 4096 × 10000 ≈ 40MB。

![图4.9.3 直接 IO 测试结果 通过 time 命令测试](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-9%20%E5%9B%BE4.9.3%20%E7%9B%B4%E6%8E%A5%20IO%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E9%80%9A%E8%BF%87%20time%20%E5%91%BD%E4%BB%A4%E6%B5%8B%E8%AF%95.png)

`time` 测得约 **2.7 秒**。

#### (3) 普通 I/O 对比

只把 `O_DIRECT` 去掉、去掉对齐修饰，其它完全不变：

```c
static char buf[8192];

fd = open("./test_file", O_WRONLY | O_CREAT | O_TRUNC, 0664);
```

![图4.9.4 普通 IO 测试结果](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-10%20%E5%9B%BE4.9.4%20%E6%99%AE%E9%80%9A%20IO%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

`time` 测得约 **0.13~0.14 秒**，比直接 I/O 快 **20 倍左右**。

原因：直接 I/O 每次 `write` 都直接对磁盘发起写；普通方式只是把用户空间数据拷进内核缓冲区，不碰硬件。硬件操作耗时远大于内存拷贝。

⚫ 所以：`O_DIRECT` 用来**测磁盘真实读写速率**才准确，日常读写千万别用。

### 4.9.5 stdio 缓冲

标准 I/O 虽是文件 I/O 的封装，但效率更高，原因就是它**自己维护了 stdio 缓冲区**。

两者的层级关系：

| 缓冲区 | 所在空间 | 维护者 | 刷新方式 |
|---|---|---|---|
| **stdio 缓冲区** | **用户空间** | 标准 I/O 库 | 缓冲满/遇 `\n`/`fflush`/`fclose`/程序退出 |
| **内核缓冲区** | **内核空间** | 内核 | `fsync`/`fdatasync`/`sync`、`O_SYNC`/`O_DSYNC`、内核自行调度 |

标准 I/O 的做法：把用户写入/读取的数据先缓存在 stdio 缓冲区，再**一次性**通过系统调用送进内核缓冲区（或从内核缓冲区拷到应用 buf）。这样**减少系统调用次数**，编程者也不必自己管缓冲。

**完整数据通路（自下而上）**：磁盘 →（内核发起 I/O）→ 内核缓冲区 →（`read`/`write` 系统调用）→ stdio 缓冲区 →（库函数）→ 应用程序 buf。

#### (1) setvbuf

```c
#include <stdio.h>

int setvbuf(FILE *stream, char *buf, int mode, size_t size);
```

| 参数 | 含义 |
|---|---|
| `stream` | FILE 指针，每个文件可单独设置自己的 stdio 缓冲区 |
| `buf` | 非 NULL 时，`buf` 指向的 `size` 字节内存作为该文件的 stdio 缓冲区。**必须动态（堆，`malloc`）或静态分配**，不能是栈上的局部变量。为 NULL 时由 stdio 库自动分配（除非 mode 是无缓冲） |
| `mode` | 缓冲类型（见下表） |
| `size` | 缓冲区大小 |

`mode` 三种取值：

| mode | 类型 | 行为 |
|---|---|---|
| `_IONBF` | **无缓冲** | 每个标准 I/O 函数立即调用 `write`/`read`，忽略 `buf`、`size`（可传 NULL 和 0）。**stderr 默认此模式**，保证错误信息立即输出 |
| `_IOLBF` | **行缓冲** | 遇换行符 `\n` 才执行文件 I/O。输出流：遇 `\n` 前缓存，遇 `\n` 时把整行通过 `write` 刷入内核缓冲区；输入流：每次读一行。**终端设备默认此模式**，如 stdin、stdout |
| `_IOFBF` | **全缓冲** | 填满 stdio 缓冲区后才执行 I/O。输出流：`fwrite` 写满缓冲区才调 `write`；输入流：每次读一个缓冲区大小的字节。**磁盘上的普通文件默认此模式** |

**返回值**：成功 0；失败返回**非 0 值**并设置 `errno`。

⚫ 数据被刷入内核缓冲区或被读走后，就**不再存在于 stdio 缓冲区**了。
⚫ 缓冲区不能给栈上的局部变量——函数返回后这块内存就失效了，而 stdio 库还在往里写。

#### (2) setbuf

```c
#include <stdio.h>

void setbuf(FILE *stream, char *buf);
```

构建于 `setvbuf` 之上，**无返回值**，等价于：

```c
setvbuf(stream, buf, buf ? _IOFBF : _IONBF, BUFSIZ);
```

`buf` 为 NULL 表示无缓冲；否则须指向调用者分配的 **BUFSIZ** 字节空间（`BUFSIZ` 在 `<stdio.h>`，通常 8192）。

#### (3) setbuffer

```c
#include <stdio.h>

void setbuffer(FILE *stream, char *buf, size_t size);
```

类似 `setbuf`，但可**自定缓冲区大小**，无返回值，等价于：

```c
setvbuf(stream, buf, buf ? _IOFBF : _IONBF, size);
```

### 4.9.6 标准输出的行缓冲测试

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("Hello World!\n");     /* 有换行 → 立即刷出 */
    printf("Hello World!");       /* 无换行 → 留在 stdio 缓冲区 */

    for ( ; ; ) sleep(1);         /* 死循环，防止程序退出时刷新 */
}
```

![图4.9.5 运行结果 运行之后](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-11%20%E5%9B%BE4.9.5%20%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C%20%E8%BF%90%E8%A1%8C%E4%B9%8B%E5%90%8E.png)

只显示第一个 "Hello World!"。原因：stdout 默认**行缓冲**，第二个 `printf` 没有 `\n`，数据仍在 stdio 缓冲区里等着。

⚫ 这也解释了 `scanf` 为什么必须按回车才继续：标准输入也是行缓冲。

改成**无缓冲**：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    /* 将标准输出设置为无缓冲模式 */
    if (setvbuf(stdout, NULL, _IONBF, 0)) {
        perror("setvbuf error");
        exit(0);
    }

    printf("Hello World!\n");
    printf("Hello World!");

    for ( ; ; ) sleep(1);
}
```

![图4.9.6 无缓冲标准输出测试结果](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-12%20%E5%9B%BE4.9.6%20%E6%97%A0%E7%BC%93%E5%86%B2%E6%A0%87%E5%87%86%E8%BE%93%E5%87%BA%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png)

两个都输出了，且光标停在第二个后面（因为没有换行）。

### 4.9.7 刷新 stdio 缓冲区

#### (1) fflush

```c
#include <stdio.h>

int fflush(FILE *stream);
```

强制把指定文件的 stdio **输出**缓冲区中的数据通过 `write()` 写入内核缓冲区。

| 参数 | 含义 |
|---|---|
| `stream` | 要刷新的文件；**传 NULL 表示刷新所有 stdio 缓冲区** |

**返回值**：成功 0，失败 -1 并设置 `errno`。

```c
printf("Hello World!\n");
printf("Hello World!");
fflush(stdout);            /* 刷新标准输出 stdio 缓冲区 */

for ( ; ; ) sleep(1);
```

![图4.9.7 刷新缓冲区](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-13%20%E5%9B%BE4.9.7%20%E5%88%B7%E6%96%B0%E7%BC%93%E5%86%B2%E5%8C%BA.png)

#### (2) 关闭文件时自动刷新

```c
printf("Hello World!\n");
printf("Hello World!");
fclose(stdout);            /* 关闭时自动刷新该文件的 stdio 缓冲区 */

for ( ; ; ) sleep(1);
```

结果与上图一致 → **`fclose()` 关闭文件时会自动刷新该文件的 stdio 缓冲区**。

#### (3) 程序退出时自动刷新

```c
int main(void)
{
    printf("Hello World!\n");
    printf("Hello World!");
}                          /* 无 return、无死循环，程序自然结束 */
```

![图4.9.8 测试结果 从结果](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-14%20%E5%9B%BE4.9.8%20%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C%20%E4%BB%8E%E7%BB%93%E6%9E%9C.png)

两个都输出了 → **程序退出时会自动刷新 stdio 缓冲区**。

⚫ 但**与退出方式有关**：用 `exit()`、`return` 或自然结束，都会刷新；用 `_exit()` 或 `_Exit()` 终止则**不会刷新**（第 3 章埋的伏笔在这里兑现）。
⚫ 所以测试缓冲行为时要在末尾加死循环，否则程序退出会掩盖现象。

**刷新 stdio 缓冲区三种方式**：

| 方式 | 说明 |
|---|---|
| `fflush(stream)` | 强制刷新指定文件（传 NULL 刷全部） |
| `fclose(stream)` | 关闭文件时自动刷新 |
| 程序退出 | `exit()`/`return`/自然结束会刷新；`_exit()`/`_Exit()` **不会** |

### 4.9.8 I/O 缓冲小结

![图4.9.9 IO 缓冲小结](photo/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%A0%87%E5%87%86IO%E5%BA%93/4-15%20%E5%9B%BE4.9.9%20IO%20%E7%BC%93%E5%86%B2%E5%B0%8F%E7%BB%93.jpg)

自上而下的数据流：

1. 应用调用**标准 I/O 库函数** → 数据进入 **stdio 缓冲区**（用户空间，stdio 库维护）；
2. 满足缓冲模式条件时，stdio 库调用**文件 I/O（系统调用）** → 数据进入**内核缓冲区**（内核空间）；
3. 最终由内核向磁盘发起读写 → 数据落盘（或从盘读入）。

可用的控制手段：

| 目标 | 手段 |
|---|---|
| 设置 stdio 缓冲模式/大小/自定义缓冲区 | `setvbuf`、`setbuf`、`setbuffer` |
| 强制刷新 stdio 缓冲区 | `fflush` |
| 刷新内核缓冲区 | `fsync`、`fdatasync`、`sync`，或 `open` 时 `O_SYNC`/`O_DSYNC` |
| 绕过内核缓冲区 | `open` 时 `O_DIRECT` |

---

## 4.10 文件描述符与 FILE 指针互转

同一个文件上可以**混用**文件 I/O 与标准 I/O，此时需要在 fd 与 FILE\* 之间转换，靠 `fileno()` 和 `fdopen()`。

```c
#include <stdio.h>

int fileno(FILE *stream);
FILE *fdopen(int fd, const char *mode);
```

| 函数 | 方向 | 说明 |
|---|---|---|
| `fileno` | FILE\* → fd | 由 FILE 指针得到整数 fd。**返回值**即 fd；转换错误返回 -1 并设置 `errno`。得到后可继续用 `read`/`write`/`lseek`/`fcntl` |
| `fdopen` | fd → FILE\* | 给定 fd 得到对应 FILE 指针，之后可用 `fread`/`fwrite`。`mode` 与 `fopen` 相同（见表 4.4.1）；**若 mode 与 fd 的访问模式不一致，`fdopen` 调用失败** |

⚫ 混合使用时**必须注意缓冲**：文件 I/O 直接写内核缓冲区，标准 I/O 先写 stdio 缓冲区、稍后才 `write` 到内核缓冲区。两者顺序不保证一致。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("print");                        /* 进 stdio 缓冲区，无换行不刷 */
    write(STDOUT_FILENO, "write\n", 6);     /* 直接进内核缓冲区，立即输出 */
    exit(0);                                /* 退出时才把 "print" 刷出 */
}
```

输出顺序是 **先 `write` 后 `print`**：`write` 立刻到内核缓冲区并显示，而 `print` 还躺在 stdio 缓冲区里，直到 `exit()` 才刷出。

⚫ 结论：混合使用标准 I/O 与文件 I/O 时，在切换之前先 `fflush()`，或者干脆只用一种。

---

## 本章速记

| 要点 | 结论 |
|---|---|
| 标准 I/O 本质 | 标准 C 库函数，封装 `open`/`read`/`write`；可移植性好、带用户空间缓冲，效率高于直接系统调用 |
| 操作对象 | 文件 I/O 用 `int fd`；标准 I/O 用 `FILE *`，作用等价 |
| 三个标准流 | stdin(0)/stdout(1)/stderr(2)；`STDIN_FILENO` 等宏在 `<unistd.h>`，`stdin` 等在 `<stdio.h>` |
| fopen | `FILE *fopen(const char *path, const char *mode)`；失败返回 **NULL** |
| mode | `r`/`r+`/`w`/`w+`/`a`/`a+`；`w` 类会**截断为 0**，`a` 类为 `O_APPEND` 追加 |
| 新建权限 | `fopen` 无法指定，固定 0666，再受 umask 削减；要精确控制用 `open` |
| fclose | 成功 0，失败 **EOF(-1)**；关闭时**自动刷新** stdio 缓冲区 |
| fread/fwrite | `(ptr, size, nmemb, stream)`，返回**数据项个数**（非字节数）；短返回要区分错误/EOF |
| fseek | 成功返回 **0**（不是偏移量！），失败 -1 |
| ftell | 返回当前偏移量；`fseek(END)` + `ftell` 可求文件大小 |
| feof/ferror | 标志置位返回**非零**；`fread` 短返回必须先查 `ferror` 再认 `feof` |
| clearerr | `void`，清两种标志；标志是粘滞的，`fseek` 成功也会清 EOF |
| 格式化输出五函数 | `printf`/`fprintf`(FILE*)/`dprintf`(fd)/`sprintf`(buf,危险)/`snprintf`(buf+size,安全)；返回字符数，失败负值 |
| format 结构 | `%[flags][width][.precision][length]type`，只有 `%` 和 `type` 必须 |
| 常用 type | `d/i` int、`o/u/x/X` unsigned、`f/F` double(默认 6 位小数)、`e/E` 科学计数、`g/G` 最短、`s` 字符串、`c` 字符、`p` 指针 |
| flags | `#` 加前缀/强制小数点、`0` 补零、`-` 左对齐(覆盖 0)、空格 正数加空格、`+` 全带符号(覆盖空格) |
| width vs precision | width 是整个输出**最小宽度**；precision 对整型是**数字个数**、对浮点是**小数位数**、对 `s` 是**最大字符数** |
| 格式化输入三函数 | `scanf`(stdin)/`fscanf`(FILE*)/`sscanf`(str)；返回**成功匹配项数**，早失败可能为 0，错误为负 |
| scanf 修饰 | `*` 读后丢弃（不传指针）；`m` 自动分配缓冲（限 `%s`/`%c`/`%[`，用完 `free`）；`width` 最大字符宽度 |
| scanf 易错 | 必须传**地址**；`%f`→float、`%lf`→double（与 printf 相反）；`%s` 不读空白；`%c` 不跳空白 |
| 内核缓冲 | `write` 只拷到内核缓冲区即返回，落盘时机不确定；减少磁盘操作次数 |
| 刷新内核缓冲 | `fsync`(数据+元数据)、`fdatasync`(仅数据)、`sync`(全部文件)；`O_SYNC`/`O_DSYNC` 等效每次 write 都刷 |
| 直接 I/O | `O_DIRECT` 绕过内核缓冲；需 `_GNU_SOURCE`；三对齐（缓冲区地址/偏移量/数据长度均为块大小整数倍）否则 EINVAL |
| 直接 I/O 性能 | 比普通 I/O 慢约 20 倍（实测 2.7s vs 0.13s），只用于测磁盘速率 |
| stdio 缓冲 mode | `_IONBF` 无缓冲（stderr 默认）、`_IOLBF` 行缓冲（终端默认）、`_IOFBF` 全缓冲（磁盘文件默认） |
| 设置 stdio 缓冲 | `setvbuf`（可设模式/大小/缓冲区）、`setbuf`（等价 `setvbuf`+BUFSIZ，无返回值）、`setbuffer`（可定大小） |
| 刷新 stdio 缓冲 | `fflush`（NULL 刷全部）、`fclose`、程序退出（`exit`/`return` 会刷，`_exit`/`_Exit` **不刷**） |
| fd ↔ FILE\* | `fileno(FILE*)` 得 fd；`fdopen(fd, mode)` 得 FILE\*，mode 必须与 fd 模式一致 |
| 混合编程 | stdio 数据可能还没落到内核缓冲，顺序会乱；切换前先 `fflush` |
