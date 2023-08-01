根据 AN13369 文档,在 psplash 代码中需要做以下修改来添加启动时间测量用的第三个脉冲:

1. 修改 Makefile.am,添加对 libgpiod 库的链接支持:

```
AM_CFLAGS += -lgpiod
GCC_FLAGS += -lgpiod
LD_FLAGS += -lgpiod
```

2. 在 psplash.c 源码头部添加 libgpiod 的头文件:

```c
#include <gpiod.h>
```

3. 在 main() 函数中添加 GPIO 初始化代码:

```c
struct gpiod_chip *chip;
struct gpiod_line *line;

chip = gpiod_chip_open("/dev/gpiochipx"); 
line = gpiod_chip_get_line(chip, 线编号);

gpiod_line_request_output(line, "psplash", 0);
gpiod_line_set_value(line, 1); 
gpiod_line_set_value(line, 0);

gpiod_line_release(line);
gpiod_chip_close(chip);
```

4. 把上述GPIO控制代码放在第一条绘图命令之前,也就是清空屏幕之前。

这样就可以在 psplash 首次绘制开机画面之前生成一个高低电平脉冲,从而测量启动时间。

这些修改需要提交补丁到 Yocto  recipes-core/psplash 目录中,然后重新编译 psplash 和镜像。


# 2

好的,根据AN13369文档,在i.MX93上实现启动时间测量的详细执行过程如下:

1. 选择GPIO pine脚

参考i.MX93的芯片手册,选择一个未被使用的GPIO脚,例如GPIO1_IO03,用于输出测量信号。

2. 更新U-Boot设备树

在arch/arm/dts/imx93-pinfunc.h中添加GPIO1_IO03的复用宏定义。

在imx93-evk.dts中禁用原先占用该脚的外设。

在iomux节点添加GPIO1_IO03的复用配置。

3. U-Boot添加第一个脉冲

在board/freescale/imx93_evk/spl.c的board_init_f函数中添加:

```
gpio_direction_output(GPIO1_IO03, 1);
gpio_direction_output(GPIO1_IO03, 0);
```

4. U-Boot添加第二个脉冲

在include/configs/imx93_evk.h的环境变量中添加loadimage命令:

```
"gpio set GPIO1_IO03; run loadimage; gpio clear GPIO1_IO03;"
```

5. psplash添加第三个脉冲 

修改recipes-core/psplash/psplash_git.bbappend,添加libgpiod支持。

修改psplash.c,在首次绘图前输出GPIO脉冲。

6. 测量启动时间

使用逻辑分析仪探针连接GPIO1_IO03和复位信号。

上电启动开发板,记录三个脉冲的时序,计算启动时间。

以上是在i.MX93上根据AN13369实现测量启动时间的基本过程。实际实现时需要参考开发板的特定硬件设计做细节调整。


# 3

好的,我来尽量详细的步骤说明一下在i.MX93上根据AN13369文档实现启动时间测量的过程:

1. 选择GPIO脚

- 查阅i.MX93芯片参考手册,找到扩展接口J18上的GPIO1_IO03脚,该脚复用功能是GPIO,可以用于输出信号。

2. 更新U-Boot设备树

- 在arch/arm/dts/imx93-pinfunc.h文件中,找到GPIO1_IO03的复用宏定义:

`#define MX93_PAD_GPIO1_3__GPIO1_3 0x17`

- 在arch/arm/dts/imx93-evk.dts文件中,注释掉原先使用该脚的外设:

`ecspi2: &ecspi2 {status = "disabled";}`

- 在iomux节点添加GPIO1_IO03的配置:

`pinctrl_hog: hoggrp {fsl,pins = <MX93_PAD_GPIO1_3__GPIO1_3 0x17>;};`

3. U-Boot添加第一个脉冲

- 在board/freescale/imx93_evk/spl.c文件中board_init_f函数后面添加:

`gpio_direction_output(IMX_GPIO_NR(1, 3), 1);`

`gpio_direction_output(IMX_GPIO_NR(1, 3), 0);`

4. U-Boot添加第二个脉冲

- 在include/configs/imx93_evk.h文件中的环境变量中,修改loadimage命令:

`"gpio set GPIO1_3; run loadimage; gpio clear GPIO1_3;\0"`

5. psplash添加第三个脉冲

- 在recipes-core/psplash/psplash_git.bbappend文件中添加:

`DEPENDS += "libgpiod"`

- 在psplash.c中包含头文件:#include <gpiod.h>

- 在main函数前面添加:

`gpiod_chip_open();`

`gpiod_line_request_output();` 

`gpiod_line_set_value(1);`

`gpiod_line_set_value(0);`

`gpiod_line_release();`

`gpiod_chip_close();`

6. 测量启动时间

- 使用逻辑分析仪连接GPIO1_IO03和nRESET信号。

- 上电开发板,在psplash消失后按复位键,记录三个脉冲时间。

- 计算从nRESET脉冲到第三个脉冲的时间差即为启动时间。

以上步骤尽可能详细的说明了在i.MX93平台使用AN13369文档实现启动时间测量的方法。实际实现时可能需要调整适配具体的开发板设计。

# 设备树前五个值

这句话的意思是,在 U-Boot 和 Linux 的设备树中,仅仅使用复用功能的复用宏还不够,需要一个额外的第六个值来表示 pad 的配置。

举个例子,在 i.MX8M 的引脚复用头文件中,定义一个引脚的复用功能的宏类似如下:

```
#define IMX8M_PAD_UART2_TXD__UART2_DCE_TX 0x21C 0x360 0x5F8 0x0 0x0
```

这里前五个值表示:
- IOMUXC 寄存器的基地址
- Pad控制寄存器的偏移地址
- Mux控制寄存器的偏移地址 
- 复用功能选择
- 复用功能内部值

但是根据这个宏定义还不足以完成正确的Pad配置,还需要一个额外的值来表示Pad控制寄存器的配置值。

所以在设备树中需要这样使用:

```
IMX8M_PAD_UART2_TXD__UART2_DCE_TX 0x21C 0x360 0x5F8 0x0 0x0 0x70
```

增加了第六个值0x70表示Pad控制寄存器的配置。

之所以需要额外这个值,是因为同一个复用功能,其Pad可根据需求配置成不同的模式,如驱动能力、上下拉、开漏等。

所以宏定义中必须包括该配置值,才能完成复用功能以及Pad配置,达到一个“adequate pin mux setup”。


是的,在 U-Boot 和 Linux 的设备树中配置 GPIO 引脚复用时,为了完成准确的引脚多路复用设置,需要提供第六个值来设置 PAD 控制寄存器的值。

这个第六个值根据所选择的 GPIO 引脚不同,需要参考芯片的硬件参考手册来决定。主要的设置考虑是:

1. 驱动能力:选择合适的驱动能力,一般选择相对大的驱动能力,以便输出稳定的高低电平。

2. 上拉/下拉:GPIO 输出模式一般关闭上拉下拉。

3. 开漏输出:关闭开漏输出,使用推挽输出。

4. 其它控制位:如速率控制、过滤器控制等根据需求设定。

例如,对于 i.MX 8M 平台,PAD 控制寄存器配置的值可以参考手册的电气特性一章的说明来选择。

对于启动时间测量使用的 GPIO,可以选择较大的驱动能力,关闭上拉下拉,配置为推挽输出模式。最终的控制值可以经过测试选择一个稳定的设置。

在 U-Boot 或者 Linux 代码中,可以提供一个复用功能配置的头文件,来定义不同 GPIO 所需的 PAD 控制值。然后在设备树中调用即可。 

