# 第二十四章 PWM 应用编程

> 本章讲 PWM 设备的应用编程。与 LED 一样，PWM 也是通过 **sysfs** 方式操控：`/sys/class/pwm/` 下导出、写属性文件即可，不需要任何库。难点在开发板出厂系统把 **PWM1 占用作 LCD 背光**，其它 PWM 因 I/O 资源不够都没引出——所以本章还演示了一个实用技巧：**在 u-boot 里动态修改内存中的设备树**，禁用 LCD/backlight 把 PWM1 腾出来测试，无需重新编译 dtb。

## 24.1 应用层如何操控 PWM

`/sys/class/pwm` 目录下有 8 个 `pwmchipX`（X=0~7）文件夹，对应 **I.MX6U 的 8 个 PWM 控制器**（可查 I.MX6U 参考手册；pwmchip0 ↔ PWM1、pwmchip1 ↔ PWM2，以此类推）：

![图24.1.1 sysclasspwm 目录下的内容](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-1%20%E5%9B%BE24.1.1%20sysclasspwm%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E5%86%85%E5%AE%B9.png)

进入 `pwmchip0` 目录：

![图24.1.2 pwmchip0目录下的内容](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-2%20%E5%9B%BE24.1.2%20pwmchip0%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E5%86%85%E5%AE%B9.png)

重点关注三个属性文件：

- **`npwm`（只读）**：该控制器下共有几路 PWM 输出。

  ![图24.1.3 读取 npwm 属性文件](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-3%20%E5%9B%BE24.1.3%20%E8%AF%BB%E5%8F%96%20npwm%20%E5%B1%9E%E6%80%A7%E6%96%87%E4%BB%B6.png)

  I.MX6U 每个 PWM 控制器只有 **1 路**输出，共 8 路（PWM1~PWM8）。**出厂系统中 PWM1 已被用作 LCD 背光**，应用层不能直接控制；其它 PWM 均不可用——I/O 资源不够，为满足其它外设的引脚需求只能如此取舍。

- **`export`（只写）**：使用前先导出（与 GPIO 的 export 一样）：

  ```bash
  echo 0 > export
  ```

  ![图24.1.4 导出 PWM](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-4%20%E5%9B%BE24.1.4%20%E5%AF%BC%E5%87%BA%20PWM.png)

  注意：**每个 pwmchipX 下导出编号都从 0 开始**；因为每个控制器只有一路，所以只能写 0：

  ```bash
  echo 0 > /sys/class/pwm/pwmchip0/export   # 导出 PWM1
  echo 0 > /sys/class/pwm/pwmchip1/export   # 导出 PWM2
  # ... pwmchip2~5 对应 PWM3~PWM7
  ```

  导出成功后会在 pwmchipX 目录下生成 **`pwm0`** 目录。

- **`unexport`（只写）**：用完删除导出的 PWM，写入的编号与 export 相对应：

  ```bash
  echo 0 > unexport
  ```

> export 和 unexport 都是**只写、没有读权限**的。

### pwm0 目录：控制 PWM

导出后进入 `pwm0` 目录，重点关注 4 个属性文件：

![图24.1.5 pwm0 目录下的内容](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-5%20%E5%9B%BE24.1.5%20pwm0%20%E7%9B%AE%E5%BD%95%E4%B8%8B%E7%9A%84%E5%86%85%E5%AE%B9.png)

| 属性文件 | 权限 | 说明 |
|---|---|---|
| `enable` | 读写 | 写 "0" 禁止 PWM、写 "1" 使能 PWM；读可查当前状态。**通常配置好后再使能** |
| `polarity` | 读写 | 极性：`normal` 普通 / `inversed` 反转。**很多 SoC 的 PWM 硬件不支持极性配置**，驱动未实现此接口——ALPHA/Mini 出厂系统便如此，写不进去 |
| `period` | 读写 | PWM 周期，**单位 ns（纳秒）**，写字符串数字 |
| `duty_cycle` | 读写 | 占空比（一个周期内高电平时间），**单位 ns** |

```bash
echo 0 > enable          # 禁止 PWM 输出
echo 1 > enable          # 使能 PWM 输出
echo normal > polarity   # 默认极性（出厂系统不支持）
echo inversed > polarity # 极性反转（出厂系统不支持）
echo 10000 > period      # 周期 10us（10000ns）
echo 5000 > duty_cycle   # 占空比 5us
```

## 24.2 编写应用程序

例程：光盘 `11、Linux C 应用编程例程源码 -> 24_pwm -> pwm.c`。执行时传 3 个参数：**编号（0=PWM1、1=PWM2…）、周期（ns）、占空比（ns）**：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

static char pwm_path[100];

/* 向指定属性文件写入字符串 */
static int pwm_config(const char *attr, const char *val)
{
    char file_path[100];
    int len, fd;

    sprintf(file_path, "%s/%s", pwm_path, attr);
    if (0 > (fd = open(file_path, O_WRONLY))) {
        perror("open error");
        return fd;
    }

    len = strlen(val);
    if (len != write(fd, val, len)) {
        perror("write error");
        close(fd);
        return -1;
    }

    close(fd);
    return 0;
}

int main(int argc, char *argv[])
{
    /* 校验传参：<id> <period> <duty> */
    if (4 != argc) {
        fprintf(stderr, "usage: %s <id> <period> <duty>\n", argv[0]);
        exit(-1);
    }

    printf("PWM config: id<%s>, period<%s>, duty<%s>\n",
           argv[1], argv[2], argv[3]);

    /* 拼出 pwm0 目录路径，如 /sys/class/pwm/pwmchip0/pwm0 */
    sprintf(pwm_path, "/sys/class/pwm/pwmchip%s/pwm0", argv[1]);

    /* 导出 pwm：pwm0 目录不存在才导出 */
    if (access(pwm_path, F_OK)) {

        char temp[100];
        int fd;

        sprintf(temp, "/sys/class/pwm/pwmchip%s/export", argv[1]);
        if (0 > (fd = open(temp, O_WRONLY))) {
            perror("open error");
            exit(-1);
        }

        if (1 != write(fd, "0", 1)) {   // 导出 pwm
            perror("write error");
            close(fd);
            exit(-1);
        }
        close(fd);
    }

    /* 配置 PWM 周期、占空比，最后使能 */
    if (pwm_config("period", argv[2]))
        exit(-1);
    if (pwm_config("duty_cycle", argv[3]))
        exit(-1);
    pwm_config("enable", "1");

    exit(0);
}
```

要点：

- `pwm_config()` 是通用写属性模板——拼路径、`O_WRONLY` 打开、`write` 字符串、关闭；
- 先 `access(pwm_path, F_OK)` 判断 `pwm0` 目录是否存在：存在=已导出，不存在才写 `export`；
- 配置顺序：**period → duty_cycle → enable**。
- 板端执行示例：

```bash
./testApp 0 500000 250000    # PWM1，周期 500us，占空比 50%
```

编译：

![图24.2.1 编译应用程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-6%20%E5%9B%BE24.2.1%20%E7%BC%96%E8%AF%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F.png)

## 24.3 在开发板上测试

编译产物拷到开发板 `/home/root`：

![图24.3.1 将测试程序拷贝到开发板homeroot目录](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-7%20%E5%9B%BE24.3.1%20%E5%B0%86%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F%E6%8B%B7%E8%B4%9D%E5%88%B0%E5%BC%80%E5%8F%91%E6%9D%BFhomeroot%E7%9B%AE%E5%BD%95.png)

### 腾出 PWM1 的两种方法

- **正规方法**：修改内核设备树——把 LCD 和 backlight 节点的 `status` 属性设为 `disabled`，重新编译设备树，用新 dtb 替换启动文件中的 dtb；或参考《I.MX6U 嵌入式 Linux 驱动开发指南》第七十三章自行配置 PWM。
- **简单方法（原书推荐，不用重编 dtb）**：u-boot 启动时会把内核设备树（dtb）拷到内存，拷进内存后就可以用 u-boot 命令**直接查看/修改**。

### u-boot 动态修改设备树

重启开发板，敲任意键进入 **u-boot 命令行模式**：

![图24.3.2 进入到 u-boot 命令行模式](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-8%20%E5%9B%BE24.3.2%20%E8%BF%9B%E5%85%A5%E5%88%B0%20u-boot%20%E5%91%BD%E4%BB%A4%E8%A1%8C%E6%A8%A1%E5%BC%8F.png)

**SD/eMMC 启动方式**执行两条命令：

```bash
setenv disable_lcd 'fdt addr ${fdt_addr}; fdt set /backlight status disable; fdt set /soc/aips-bus@02100000/lcdif@021c8000 status disable'

setenv mmcboot 'echo Booting from mmc ...; run mmcargs; if test ${boot_fdt} = yes || test ${boot_fdt} = try; then if run loadfdt; then run disable_lcd; bootz ${loadaddr} - ${fdt_addr}; else if test ${boot_fdt} = try; then bootz; else echo WARN: Cannot load the DT; fi; fi; else bootz; fi;'
```

**NAND 启动方式**：

```bash
setenv disable_lcd 'fdt addr ${fdt_addr}; fdt set /backlight status disable; fdt set /soc/aips-bus@02100000/lcdif@021c8000 status disable'

setenv bootcmd 'nand read ${loadaddr} 0x620000 0x800000;nand read ${fdt_addr} ${fdt_offset} 0x20000; run disable_lcd; bootz ${loadaddr} - ${fdt_addr}'
```

两条命令的作用：`disable_lcd` 环境变量里用 `fdt set` 把 `/backlight` 和 LCD 控制器（lcdif@021c8000）的 status 改成 disable；`mmcboot`/`bootcmd` 在加载 dtb 之后、启动内核之前先 `run disable_lcd`。拷贝时注意格式（原书 PDF 提取版里连字符可能丢失，`aips-bus` 若报错按 u-boot 提示调整）。

执行完两条 setenv 后启动：

```bash
boot
```

![图24.3.3 在 u-boot 下执行命令](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-9%20%E5%9B%BE24.3.3%20%E5%9C%A8%20u-boot%20%E4%B8%8B%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4%20%E6%89%A7%E8%A1%8C%E5%AE%8C%E4%B8%A4%E6%9D%A1%E5%91%BD%E4%BB%A4%E5%90%8E%E6%8E%A5.png)

![图24.3.4 启动内核 Tips](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-10%20%E5%9B%BE24.3.4%20%E5%90%AF%E5%8A%A8%E5%86%85%E6%A0%B8%20Tips.png)

> **Tips**：这种修改**只对本次启动生效**——改的是内存里那份设备树，下次重启就恢复原状。

### 运行测试、示波器验证

系统启动后 LCD 被禁用，PWM1 已腾出。执行测试程序：

![图24.3.5 运行测试程序](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-11%20%E5%9B%BE24.3.5%20%E8%BF%90%E8%A1%8C%E6%B5%8B%E8%AF%95%E7%A8%8B%E5%BA%8F.png)

本实验测 **PWM1**：出厂系统已把 PWM1 输出绑定到 **GPIO1_IO08 引脚（即 LCD 背光引脚）**，该引脚已从开发板扩展口引出：

![图24.3.6 GPIO1_IO08引脚 ALPHA板（左）](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-12%20%E5%9B%BE24.3.6%20GPIO1_IO08%E5%BC%95%E8%84%9A%20ALPHA%E6%9D%BF%EF%BC%88%E5%B7%A6%EF%BC%89.jpg)

![图24.3.6 GPIO1_IO08引脚 Mini板（右）](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-13%20%E5%9B%BE24.3.6%20GPIO1_IO08%E5%BC%95%E8%84%9A%20Mini%E6%9D%BF%EF%BC%88%E5%8F%B3%EF%BC%89.jpg)

Mini 板可通过背面丝印名称或原理图确认引脚。

用示波器测 GPIO1_IO08 的波形：

![图24.3.7 PWM波形示波器实测](photo/%E7%AC%AC%E4%BA%8C%E5%8D%81%E5%9B%9B%E7%AB%A0%20PWM%E5%BA%94%E7%94%A8%E7%BC%96%E7%A8%8B/24-14%20%E5%9B%BE24.3.7%20PWM%E6%B3%A2%E5%BD%A2%E7%A4%BA%E6%B3%A2%E5%99%A8%E5%AE%9E%E6%B5%8B.png)

实测：周期 500us（500000ns）、频率约 2KHz、占空比 49.99%——与 `./testApp 0 500000 250000` 的配置一致。

## 24.4 编译与部署（结合本机环境）

```bash
# Ubuntu 上（~/linux/IMX6ULL/app-test/24_pwm/）
arm-linux-gnueabihf-gcc pwm.c -o pwm
scp pwm root@192.168.1.104:/root/pwm     # 目标带文件名
```

```bash
# 板子上（先按 24.3 的 u-boot 方法腾出 PWM1 并重启）
./pwm 0 500000 250000        # PWM1：周期 500us、占空比 50%
cat /sys/class/pwm/pwmchip0/pwm0/period        # 验证 500000
cat /sys/class/pwm/pwmchip0/pwm0/duty_cycle    # 验证 250000
cat /sys/class/pwm/pwmchip0/pwm0/enable        # 验证 1
# 用完删除导出
echo 0 > /sys/class/pwm/pwmchip0/unexport
```

> 没有示波器也能粗验证：用万用表量 GPIO1_IO08 直流电压，占空比 50% 时约 1.6V 左右（3.3V × 50%）；改占空比重测，电压随比例变化即说明 PWM 在输出。

## 本章速记

| 要点 | 内容 |
|---|---|
| 操控方式 | sysfs：`/sys/class/pwm/pwmchipX/`，无需设备节点和库 |
| pwmchipX ↔ PWM | pwmchip0=PWM1 … pwmchip7=PWM8；`npwm` 只读，I.MX6U 每控制器 1 路 |
| 导出/删除 | `echo 0 > export` / `echo 0 > unexport`；编号每控制器都从 0 开始；两文件只写无读权限 |
| 四个属性 | `enable`(0/1)、`polarity`(normal/inversed，出厂驱动未实现)、`period`(ns)、`duty_cycle`(ns) |
| 单位 | 周期与占空比都以 **ns** 写入；10us=10000 |
| 配置顺序 | 先 period/duty_cycle，最后 `enable` 写 1 |
| 出厂限制 | PWM1 被 LCD 背光占用；其它 PWM 未引出 |
| 腾出 PWM1 技巧 | u-boot 里 `fdt addr ${fdt_addr}` + `fdt set /backlight status disable` + lcdif 节点 disable → `boot`；**只对本次启动有效** |
| 测试引脚 | PWM1 ↔ GPIO1_IO08（LCD 背光引脚），已引出到扩展口 |
| 验证 | `./pwm 0 500000 250000` → 2KHz、50% 占空比 |
