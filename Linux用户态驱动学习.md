# Linux用户态驱动学习

本文档用来记录linux对底层硬件的封装对象的学习，学习使用命令行驱动对应外设。

不同的设备被抽象成文件，存放在 `/dev` 目录下供用户进程进行操作。用户通过这些设备文件，可以实现对硬件进行相应的操作。而这些设备文件，需要由对应的设备文件系统来负责管理。在 kernel 2.6 之前，完成这一使命的是 devfs。devfs 是 Linux 2.4 引入的一个虚拟的文件系统，挂载在 `/dev` 目录下。可以动态地为设备在 `/dev` 下创建或删除相应的设备文件，只生成存在设备的节点。在kernel 2.6 之后，就引入了sysfs来操作设备文件，sysfs 是一个基于内存的虚拟的文件系统，由 kernel 提供，挂载到`/sys` 目录下，负责以设备树的形式向用户提供直观的设备和驱动信息，sysfs更像树状结构，可以查设备名字、地址等等。现在`/dev` `/sys`这两个都能用，但是常用的是sysfs。

[sysfs、udev 和 它们背后的 Linux 统一设备模型 - 博客 - binsite](https://www.binss.me/blog/sysfs-udev-and-Linux-Unified-Device-Model/)

## GPIO

对于sysfs，gpio对象被封装在/sys/classs/gpio里面，里面包含了gpio组的控制器，其文件夹内容如下：

```
[root@qemu_imx6ul:/sys/class/gpio]# ls
export       gpiochip128  gpiochip64   unexport
gpiochip0    gpiochip32   gpiochip96
```

可以看到，包含了5组gpio控制器，每个控制器控制32个gpio引脚（即芯片的IO端口有5个，如A,B,C,D,E，然后每组有32个引脚），假如我要控制gpioB20，其对应的就是（1*32+20=52），即是需要控制的引脚为gpio52，也可以直接进入控制器文件夹，查看控制器的基地址，加上偏移量也可以。

```
[root@qemu_imx6ul:/sys/class/gpio]# cd gpiochip32
[root@qemu_imx6ul:/sys/class/gpio/gpiochip32]# ls
base       device     label      ngpio      power      subsystem  uevent
[root@qemu_imx6ul:/sys/class/gpio/gpiochip32]# cat base 
32
```

- 首先需要确定要控制的gpio引脚，将其导出为可使用

```
[root@qemu_imx6ul:/sys/class/gpio]# echo 52 > export 
[root@qemu_imx6ul:/sys/class/gpio]# ls
export       gpiochip0    gpiochip32   gpiochip96
gpio52       gpiochip128  gpiochip64   unexport
```

在linux里，所以设备都可以看作一个文件，所以只需要使用echo向该文件写入，即可完成对应的操作，使用`echo 52 > export`将gpioB20导出后，可以看到生成了对应的文件夹gpio52

- 进入该文件夹，可以看到对应的一些操作文件

```
[root@qemu_imx6ul:/sys/class/gpio]# cd gpio52
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# ls
active_low  direction   power       uevent
device      edge        subsystem   value
```

其中：

1. direction 控制其输入输出，使用`echo in > direction``echo out > direction`可以控制该gpio的输入输出模式，使用`cat direction`可以查看目前gpio的模式
2. value 是当前gpio的电平值，使用`echo 1 > direction``echo 0 > direction`可以控制输出电平，使用`cat value`可以直接读出目前的电平
3. 其他待补充

- 如果需要取消对gpio的控制，可以写入unexport，使用`echo 52 > unexport`取消其控制

```
[root@qemu_imx6ul:/sys/class/gpio]# echo 52 > unexport  
[root@qemu_imx6ul:/sys/class/gpio]# ls
export       gpiochip128  gpiochip64   unexport
gpiochip0    gpiochip32   gpiochip96
```

取消控制之后，对应的文件夹也会消失。linux库也向C语言提供了一套用户态的操作函数`open(),write(),close()`，可以打开，关闭，写入对应文件`open(),write(),close()`，并返回对应的文件操作符号fd。这就对应了我们终端的echo，export，unexport.

### 控制一颗LED灯的亮灭

```
[root@qemu_imx6ul:/sys/class/gpio]# ls
export       gpiochip128  gpiochip64   unexport
gpiochip0    gpiochip32   gpiochip96
[root@qemu_imx6ul:/sys/class/gpio]# echo 52 > export   
[root@qemu_imx6ul:/sys/class/gpio]# ls
export       gpiochip0    gpiochip32   gpiochip96
gpio52       gpiochip128  gpiochip64   unexport
[root@qemu_imx6ul:/sys/class/gpio]# cd gpio52
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# ls
active_low  direction   power       uevent
device      edge        subsystem   value
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# cat value 
0
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# cat direction 
in
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# echo out > direction 
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# cat direction 
out
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# echo 1 > value   
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# cat value 
1
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# echo 0 > value 
[root@qemu_imx6ul:/sys/class/gpio/gpio52]# cat value 
0
```

## I2C

命令行使用I2C-toos对I2C设备进行操作，下图是内核态到用户态的整个过程。

![内核态到用户态](https://img-blog.csdn.net/20160914150243634)

对于sysfs，I2C对象被封装在/sys/classs/里面，里面可以查到I2C设备的相关属性

```
[root@qemu_imx6ul:/sys/class]# ls
ata_device    firmware      mdio_bus      rc            tty
ata_link      gpio          mem           regulator     ubi
ata_port      graphics      misc          rtc           udc
backlight     hwmon         mmc_host      scsi_device   vc
bdi           i2c-dev       mtd           scsi_disk     video4linux
block         ieee80211     net           scsi_host     vtconsole
bluetooth     input         power_supply  spi_master    watchdog
drm           lcd           pps           tee
extcon        leds          ptp           thermal
[root@qemu_imx6ul:/sys/class]# cd i2c-dev/
[root@qemu_imx6ul:/sys/class/i2c-dev]# ls
i2c-0
[root@qemu_imx6ul:/sys/class/i2c-dev]# cd i2c-0
[root@qemu_imx6ul:/sys/class/i2c-dev/i2c-0]# ls
dev        device     name       power      subsystem  uevent
```

由这个文件夹的内容就能看出，该系统挂载了一路I2C，设备名字叫做i2c-0。这里没有适合的文件给我们写入，但是我们可以直接去/dev里面，可以直接cat i2c-0就能读到数据，不过是乱码，也不知道是什么数据。我们这里使用I2C-tools工具操作I2C，可以直接显示数据。

- 首先我们读出目前IIC上所有挂载的设备地址（-y是值忽略询问 0是指总线号，上面查出来的）

```
[root@qemu_imx6ul:~]# i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --
```

可以看到有一个设备挂载在I2C的0x50地址上，接下来我们读取该设备的寄存器内容（-y是值忽略询问 0是指总线号，0x50是寄存器地址，以字节形式显示）

```
[root@qemu_imx6ul:~]# i2cdump -y 0 0x50 b
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: 11 58 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ?X..............
10: 00 11 00 00 00 00 00 00 00 00 00 00 00 00 00 00    .?..............
20: 00 00 11 00 00 00 00 00 00 00 00 00 00 00 00 00    ..?.............
30: 00 00 00 79 00 00 00 00 00 00 00 00 00 00 00 00    ...y............
40: 00 00 00 00 79 00 00 00 00 00 00 00 00 00 00 00    ....y...........
50: 00 00 00 00 00 79 00 00 00 00 00 00 00 00 00 00    .....y..........
60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
90: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
c0: 00 00 00 00 00 00 00 00 00 00 00 00 79 00 00 00    ............y...
d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 79 00 00    .............y..
e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 79 00    ..............y.
f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 79    ...............y
```

向设备对应位置上的寄存器写入0x66值，可以看到成功写入了。

```
[root@qemu_imx6ul:~]# i2cset -y 0 0x50 0x88 0x66
[root@qemu_imx6ul:~]# i2cdump -y 0 0x50 b
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: 11 58 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ?X..............
10: 00 11 00 00 00 00 00 00 00 00 00 00 00 00 00 00    .?..............
20: 00 00 11 00 00 00 00 00 00 00 00 00 00 00 00 00    ..?.............
30: 00 00 00 79 00 00 00 00 00 00 00 00 00 00 00 00    ...y............
40: 00 00 00 00 79 00 00 00 00 00 00 00 00 00 00 00    ....y...........
50: 00 00 00 00 00 79 00 00 00 00 00 00 00 00 00 00    .....y..........
60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
80: 00 00 00 00 00 00 00 00 66 00 00 00 00 00 00 00    ........f.......
90: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
c0: 00 00 00 00 00 00 00 00 00 00 00 00 79 00 00 00    ............y...
d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 79 00 00    .............y..
e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 79 00    ..............y.
f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 79    ...............y

```

tips: 实际上的设备地址为0xA0，对应的是1010 0000，但i2c使用的是7位地址，即是0101 0000，最后读出来的地址就是0x50
