# HDL通用平台架构代码学习与编译使用

> 笔者： 吴有建

## 代码架构

```shell
hdl/
├── code/
│   ├── include/          ← 每个设备的操作表，每个操作表中包含具体的操作函数，调用时会查询表中是否有对应实际案例并调用，函数名在ops表中已经确定好了
│   ├── src/              ← 应用层调用业务函数，就是调用HDL_OPS_RUN封装转发一次，HDL_OPS_RUN里面根据函数名来调用ops表内的各个操作函数
│   ├── drivers/          ← 实现ops表的操作函数，把硬件当作文件操作，定义ctx, ctx对应硬件的参数，如端口号，引脚;
│   │   └── plat_init/    ← hdl_plat_init初始化对应平台，里面调用HDL_DRV_REG将不同板子的硬件注册进g_hdl_registry数组里面，这个数组会按需扩容
│   ├── examples/         ← 示例程序
│   └── test/             ← 调试命令（在 shell 里调试）
├── plats/                ← 各板子的编译配置（用哪个交叉编译器）
├── libs/                 ← 别人写好的库（日志、shell 等，按 CPU 架构分目录）
└── third/                ← 第三方源码
```

## 初始化逻辑

###hdl_plat_init 

- 主函数运行时，首先进行的是平台初始化 **hdl_plat_init**，不同平台需要有不同的初始化，以树莓派5平台的举例，其他平台逻辑也类似
- 该函数主要是调用**HDL_DRV_REG(设备类型, 驱动名, param, 实例名, 标志)**来注册不同的硬件，不同平台都是依靠这个函数注册硬件
- 在HDL_DRV_REG宏中主要包括两个重要的函数：**gpio_led_create**和**hdl_inst_register**

```c
int32_t hdl_plat_init(const char *proc, int32_t steps)
{
    (void)proc;
    (void)steps;

    HDL_DRV_REG(HDL_INST_LED,           gpio,   17,             "status",           0);
    HDL_DRV_REG(HDL_INST_BUTTON,        gpio,   18,             "user_btn",         0);
    HDL_DRV_REG(HDL_INST_UART,          bcm,    "/dev/ttyAMA0", "console",          0);

    HDL_DRV_REG(HDL_INST_WIFI_BT,       virt,   0,              "wifi_bt0",         0);
    HDL_DRV_REG(HDL_INST_SLAVECORE,     virt,   0,              "dsp_core",         0);
    HDL_DRV_REG(HDL_INST_OTA,           virt,   0,              "sys_ota",          0);

    return EC_OK;
```

```c
HDL_DRV_REG(HDL_INST_LED,    gpio,   	  			  17,              "status",           0 );
//              ↑ 设备类型     ↑ 驱动名（不能随便填）      ↑ 参数				↑ 实例名		  ↑ 标志
```

```c
// 设备类型定义 HDL_INST_NUM占位符，代表设备总数
CMD_ENUM_DFE(hdl_inst_type_t, 0,
    HDL_INST_TEMPERATURE,      /* 温度测量 */
    HDL_INST_LED,              /* LED 指示 */
    HDL_INST_BUTTON,           /* 按键输入 */
    HDL_INST_UART,             /* 串口数据收发 */
    HDL_INST_PWM,              /* PWM 输出 */
    HDL_INST_ANALOG,           /* 模拟量采集 */
    HDL_INST_VISREC,           /* 视觉识别 */
    HDL_INST_WIFI_BT,          /* WiFi / 蓝牙协同控制 */
    HDL_INST_CAMERA,           /* 摄像头图像采集 */
    HDL_INST_SLAVECORE,        /* 从核子系统通信 */
    HDL_INST_OTA,              /* 系统固件 OTA 升级 */
    HDL_INST_CUSTOM,           /* 自定义功能 */
    HDL_INST_NUM
);
```

####HDL_DRV_REG

- 这个一个函数宏，`drv##_##type##_create` 这种写法，`##` 是 C 预处理器的**字符串拼接符**，会把碎片粘成一个完整的函数名：

```c
#define HDL_DRV_REG(type, drv, param, name, flag) \
    do { \
        int32_t drv##_##type##_create(hdl_inst_reg_t *r, void *p); \
        void drv##_##type##_destroy(hdl_inst_reg_t *r); \
        hdl_inst_reg_t __reg = {0}; \
        int __ret = drv##_##type##_create(&__reg, (void *)(param)); \
        if (__ret != EC_OK) { \
            LOG_ERROR("[hdl_plat] Failed to create %s instance '%s': %d", #type, name, __ret); \
            return __ret; \
        } \
        __ret = hdl_inst_register(&__reg, name, flag); \
        if (__ret != EC_OK) { \
            LOG_ERROR("[hdl_plat] Failed to register %s instance '%s': %d", #type, name, __ret); \
            drv##_##type##_destroy(&__reg); \
            return __ret; \
        } \
    } while (0)
```

当我们在板级文件里写：

```c
HDL_DRV_REG(HDL_INST_LED, gpio, 17, "status", 0);
```

预处理后它变成：

```c
int32_t gpio_HDL_INST_LED_create(hdl_inst_reg_t *r, void *p);   // 要调用这个函数，传参按照实际来
gpio_HDL_INST_LED_create(&__reg, (void *)17);                   // 对上实际的参数就是在调用这个函数，包括传参
```

当然这里还有一个hdl_inst_register函数，这个留到下面讲

#### HDL_DRV_CREATE_DEF

在gpio_led.c里面，对gpio_HDL_INST_LED_create这个函数做出了定义

```c
#define HDL_DRV_CREATE_DEF(itype, drv, func) \
    int32_t drv##_##itype##_create(hdl_inst_reg_t *reg, void *param) { \
        if (reg) reg->type = itype; \
        return func(reg, param); \
    }
```

```c
HDL_DRV_CREATE_DEF(HDL_INST_LED, gpio, gpio_led_create); 
```

展开后是定义了一个注册函数

```c
int32_t gpio_HDL_INST_LED_create(hdl_inst_reg_t *reg, void *param) {
    if (reg) reg->type = HDL_INST_LED;        // 顺手把类型填上
    return gpio_led_create(reg, param);       // 转调真正干活的函数
}
```

其实就是在调用驱动提供的函数

```c
板级平台文件（调用方）                     底层驱动文件（提供方）
HDL_DRV_REG(HDL_INST_LED, gpio,…)   HDL_DRV_CREATE_DEF(HDL_INST_LED, gpio, …)
        │                                      │
        └────────► gpio_HDL_INST_LED_create ◄──┘
                     两边拼出的名字必须一致
```

#### HDL_DRV_DESTROY_DEF

同理，注销函数也是一致的

```c
#define HDL_DRV_DESTROY_DEF(itype, drv, func) \
    void drv##_##itype##_destroy(hdl_inst_reg_t *reg) { \
        func(reg); \
    }

HDL_DRV_DESTROY_DEF(HDL_INST_LED, gpio, gpio_led_destroy);
```

###gpio_led_create

- 调用注册宏就是在调用注册函数，以led的作为实例，就是调用了gpio_led_create，适配硬件的操作全部发生在 create 里。

- hdl_inst_reg_t 是一个中间存储对象，获得对应的操作表和ctx后传入hdl_inst_register注册

```C
typedef struct {
    int     gpio_pin;       /* GPIO 引脚号 */
    int     fd_value;       /* /sys/class/gpio/gpioN/value */
    int     fd_direction;   /* /sys/class/gpio/gpioN/direction */
} gpio_led_ctx_t;

int32_t gpio_led_create(hdl_inst_reg_t *reg, void *param)
{
    int32_t gpio_pin = (int32_t)(intptr_t)param;  // 获得传入参数 17

    ctx = calloc(1, sizeof(gpio_led_ctx_t));      // 创建一个保存参数上下文的变量

    ctx->gpio_pin = gpio_pin;					  // 上下文保存引脚信息
    gpio_export(gpio_pin);                        // 调用linux内核 export导出17号引脚 对应终端echo 17 > export
    gpio_set_direction(gpio_pin, "out");          // 调用linux内核 设置引脚输出输入方向 对应终端echo in > direction
    ctx->fd_value = gpio_open_value(gpio_pin);    // 上下文保存打开gpio返回的文件操作符
                                                  // 通过/sys/class/gpio/gpio17/value 得到 fd

    reg->drv_ctx = ctx;                           // 把ctx存到reg中，该参数供操作表调用
    reg->ops = &gpio_led_ops;                     // 把ops表存到reg中，以后调用时可以直接ops->func(ctx)
    return EC_OK;
}
```

#### ops操作表

这个表绑定对应的函数，根据这个表就可以调用对应的函数地址，而不是过多需要暴露对应的函数的内容，以后调用时可以直接ops->func(参数)

```c
const hdl_led_ops_t gpio_led_ops = {
    .set             = gpio_led_set,
    .get             = gpio_led_get,
    .set_color       = NULL,
    .set_blink_pattern = NULL,
};
```

也就是说，上面reg已经保存了对应的函数ops表和硬件参数ctx，要使用直接调用这个reg的操作表就行。

### hdl_inst_register

实例对象结构体 hdl_instance

```c
struct hdl_instance {
    hdl_inst_type_t     type;	// 设备类型
    char                name[HDL_NAME_MAX]; //限制命名大小
    const void         *ops;//操作表
    void               *drv_ctx; //参数上下文
    sem_t              *lock_sem;   /* POSIX 命名信号量，NULL 表示不使用 */ //用来加锁的
};
typedef struct hdl_instance *hdl_inst_t;
```

这个函数负责将create返回的reg对象保存到g_hdl_registry这个数组中，根据设备类型保存到g_hdl_registry数组的对应位置，每个位置又有一个动态数组（指针）*instances，持续保存相同设备驱动的不同对象，也就是每个大格子里都有很多个实例对象，每个实例对象都是hdl_instance指针。使用时可以理解为二维数组，g_hdl_registry[type].instances[i].name 

```c
/* 按类型分组的注册表（动态分配，自动扩容） */
typedef struct {
    struct hdl_instance *instances;   指针（利用calloc初始分配内存，后续用realloc重新分配并扩大内存，内存地址会变）
    int32_t             count;        /* 当前已注册数量 */
    int32_t             capacity;     /* 当前容量 */
} hdl_registry_entry_t;

static hdl_registry_entry_t g_hdl_registry[HDL_INST_NUM]; // 12个不同设备对应的实例对象存储表
```

存储对象结构体，每一个hdl_registry_entry_t对应一个类型，拥有动态存储的数组可保存同一类型的对象。一开始时，先分配4个大小的存储空间，此时capacity为4。每保存一个，count++，当count大于capacity时，调用realloc重新2倍扩容

```c
realloc
旧块（退还系统0x1000）：
┌────┬────┬────┬────┐
│status│..│..│..│ ──内容抄走──┐
└────┴────┴────┴────┘          │
                                ▼
新块（0x2000）：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│status│..│..│..│ 空 │ 空 │ 空 │ 空 │
└────┴────┴────┴────┴────┴────┴────┴────┘
```

hdl_inst_register函数具体代码

```c
int32_t hdl_inst_register(hdl_inst_reg_t *reg, const char *inst_name, uint32_t flags)
{
    struct hdl_instance *inst;
    int32_t ret;

    // 检查有没有操作表 名字 或者 reg是不是为空
    if (!reg || !inst_name || !reg->ops) {
        LOG_ERROR("[%s] Invalid arguments", inst_name);
        return EC_INVAL;
    }

    // 检查名字有没有超长
    if (strlen(inst_name) >= HDL_NAME_MAX) {
        LOG_ERROR("[%s] Instance name too long", inst_name);
        return EC_NAMETOOLONG;
    }

    // 检查是不是声明的设备类型
    if (reg->type >= HDL_INST_NUM) {
        LOG_ERROR("[%s] Invalid type", inst_name);
        return EC_INVAL;
    }

    // 自动扩容
    /* 容量不足时自动扩容 */
    if (g_hdl_registry[reg->type].count >= g_hdl_registry[reg->type].capacity) {
        ret = hdl_registry_grow(reg->type);
        if (ret != EC_OK) {
            return ret;
        }
    }

    // 检查该设备是否已经注册了
    /* 内联检查是否已注册 */
    for (int32_t i = 0; i < g_hdl_registry[reg->type].count; i++) {
        if (strcmp(g_hdl_registry[reg->type].instances[i].name, inst_name) == 0) {
            LOG_ERROR("[%s] Already registered", inst_name);
            return EC_EXIST;
        }
    }

    // 取12格着中对应设备的动态数组，并在动态数组中找到最后的位置把新的设备添加进去，最后count++
    inst = &g_hdl_registry[reg->type].instances[g_hdl_registry[reg->type].count];

    // 填数据而不是直接拷贝赋值，全是指针操作，减少cpu的占用量
    strncpy(inst->name, inst_name, HDL_NAME_MAX - 1);
    inst->name[HDL_NAME_MAX - 1] = '\0';
    inst->type      = reg->type;
    inst->ops       = reg->ops;
    inst->drv_ctx   = reg->drv_ctx;
    inst->lock_sem  = NULL;

    g_hdl_registry[inst->type].count++;

    return EC_OK;
}
```

### 总结

初始化流程就是：

```c
hal_plat_init
->HDL_DRV_REG(xxx)
->xxx_create
->linux实际创建对应的设备，获得文件描述符fd
->返回含有ops、ctx、instance_type的reg
->hdl_inst_register
->将reg里面的数据存储到g_hdl_registry中
->使用时，执行g_hdl_registry[对应的设备类型][name].ops->func(ctx)就行
```

##平台化操作逻辑

###hdl_led_lookup

用hdl_inst_t inst = hdl_led_lookup("status");举例子，返回的就是存储数组里面的某个设备对象的实例指针

```c
static inline hdl_inst_t hdl_led_lookup(const char *inst_name)
{
    return hdl_inst_find(HDL_INST_LED, inst_name); // 去g_hdl_registry数组中查找实例指针
}
```

```c
hdl_inst_t hdl_inst_find(hdl_inst_type_t type, const char *inst_name)
{
    if (type >= HDL_INST_NUM) {
        LOG_ERROR("[%d] Invalid type", type);
        return NULL;
    }

    if (!g_hdl_registry[type].instances) {
        LOG_ERROR("[%s] Type %d not initialized", inst_name, type);
        return NULL;
    }
	
    // 根据设备名字以及设备类型查找对应的实例指针
    for (int32_t i = 0; i < g_hdl_registry[type].count; i++) {
        if (strcmp(g_hdl_registry[type].instances[i].name, inst_name) == 0) {
            return &(g_hdl_registry[type].instances[i]);
        }
    }

    LOG_ERROR("[%s] No instance found in type %d", inst_name, type);
    return NULL;
}
```

###hdl_led_set

- hdl_led_xxx的命名和操作表具有的函数有关，因为该函数的实现其实是薄转发

```c
int32_t hdl_led_set(hdl_inst_t inst, hdl_led_state_t state)
{
    return HDL_OPS_RUN(inst, hdl_led_ops_t, set, // 这里的set其实对应的就是操作表中的set函数
                       inst->drv_ctx, state);
}
```

#### HDL_OPS_RUN

这里把HDL_OPS_RUN(inst, ops_type, func_name, ...)定义成HDL_OPS_RUN_RET(__ret, inst, ops_type, func_name, __VA_ARGS__) 

实际上调用HDL_OPS_RUN就是在调用HDL_OPS_RUN_RET，他的返回值就是错误码

这里的__VA_ARGS__指代的是前面传入的...代表的所有参数，

```c
#define HDL_OPS_RUN(inst, ops_type, func_name, ...) HDL_OPS_RUN_RET(__ret, inst, ops_type, func_name, __VA_ARGS__) 
```

#### HDL_OPS_RUN_RET

调用HDL_OPS_RUN_RET，在回错误码之前，先把ops返回值放进 RET 指定的第一个变量ret里面

```c
#define HDL_OPS_RUN_RET(ret, inst, ops_type, func_name，...)\
({\                                          
    int32_t __ret = EC_OK;\                  // ┐
    if (!(inst)) {\                          // │ 句柄是空的？
        __ret = EC_NOENT;\                   // │   → 错误码"实例不存在"
    } else {\                                // │
        const ops_type *ops = (const ops_type *)(inst)->ops;\   // 在实例中提取获取操作表
        if (!ops || !ops->func_name) {\      // │ 操作表有没有这个函数
            __ret = EC_NXIO;\                // │   → 错误码"这个函数功能未实现"
        } else {\                            // │
            if (inst->lock_sem) {\           // │ 可选加锁：这个实例要求排队吗？
                sem_wait(inst->lock_sem);\   // │   拿钥匙
                ret = ops->func_name(__VA_ARGS__);\  // ★ 调函数，...里面的所有参数，传到这里
                sem_post(inst->lock_sem);\   // │   还钥匙
            } else {\                        // │
                ret = ops->func_name(__VA_ARGS__);\  // ★ 不用锁，直接调函数
            }\                               // │
        }\                                   // │
    }\                                       // ┘
    __ret;\                                  // ← 返回整个表达式的值 = 错误码
})
```

###总结

**RUN 是“调完就回错误码”，RUN_RET 是“回错误码之前，先把值放进 RET 指定的变量”**。



## 编译与adb调试

### 编译

> 需要注意的是，不要在板子上直接编译，先在外部环境编译完后，adb推送到板子上

以星宸平台为例，当需要添加一个串口设备（如uart 4)时，如果已经写过驱动了，那就需要为其创建ops表与drv_ctx，所以需要在平台初始化时加入

```c
#include "hdl_plat.h"

#include <string.h>

int32_t hdl_plat_init(const char *proc, int32_t steps)
{
    // (void)proc;
    (void)steps;

    // HDL_DRV_REG(HDL_INST_TEMPERATURE,   sunxi,  0,              "cpu_core",         0);
    // HDL_DRV_REG(HDL_INST_LED,           virt,   0,              "status",           0);
    // HDL_DRV_REG(HDL_INST_BUTTON,        virt,   0,              "user_btn",         0);
    // HDL_DRV_REG(HDL_INST_UART,          sunxi,  "/dev/ttyAS10", "chassis",          0);
    // HDL_DRV_REG(HDL_INST_PWM,           virt,   0,              "cooling_fan",      0);
    // HDL_DRV_REG(HDL_INST_ANALOG,        virt,   0,              "battery_voltage",  0);
    // HDL_DRV_REG(HDL_INST_VISREC,        shm,    0,              "obj_detect",       0);

    /* proc 为 NULL 时默认进入本分支
       SSU9383 UART4，板上节点为 /dev/ttyS4（ttyS0 为控制台） */
    if (!proc || strcmp(proc, "uart_test") == 0) {
        HDL_DRV_REG(HDL_INST_UART,          sunxi,  "/dev/ttyS4",   "uart4",            0);
        return EC_OK;
    }

    if (strcmp(proc, "uwrrd") == 0) {
        HDL_DRV_REG(HDL_INST_VISREC,        shm,    0,              "image_get",        0);
        return EC_OK;
    }

    if (strcmp(proc, "chassis_protocol") == 0) {
        HDL_DRV_REG(HDL_INST_UART,          sunxi,  "/dev/ttyAS10", "chassis",          0);
        HDL_DRV_REG(HDL_INST_VISREC,        shm,    1,              "obj_detect",       0);
        HDL_DRV_REG(HDL_INST_WIFI_BT,       virt,   0,              "wifi_bt0",         0);
        return EC_OK;
    }

    return EC_OK;
}
```

保存后，将这份代码放到拥有对应平台编译工具链的环境里面编译，在C:\Users\wuyoujian\Desktop\单目自研-sstar\Middleware\hdl\plats\ssu9383.mk里

```makefile
# --------------------- 工具链 ---------------------
CROSS_PREFIX = /opt/sigmastar_tool/gcc-10.2.1-20210303-sigmastar-glibc-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
CC       = $(CROSS_PREFIX)gcc
CFLAGS   = -g -O2 -Wall -Wextra -std=c11 -fPIC
AR       = $(CROSS_PREFIX)ar
ARFLAGS  = rcs

# --------------------- 板级驱动文件 ---------------------
PLAT_SRC    = code/drivers/plat_init/plat_ssu9383.c
PLAT_DRV_SOURCES = \
    drivers/temp/virt_temp.c \
    drivers/temp/sunxi_temp.c \
    drivers/led/virt_led.c \
    drivers/button/virt_button.c \
    drivers/uart/virt_uart.c \
    drivers/uart/sunxi_uart.c \
    drivers/pwm/virt_pwm.c \
    drivers/analog/virt_analog.c \
    drivers/visrec/shm_visrec.c \
    drivers/wifi_bt/virt_wifi_bt.c \
    drivers/camera/virt_camera.c \
    drivers/slavecore/virt_slavecore.c \
    drivers/ota/virt_ota.c

# --------------------- 依赖库目录 ---------------------
LIBS_ARCH = aikaid
TARGET_LIBS_DIR = $(LIBS_ROOT)/$(LIBS_ARCH)

# --------------------- 额外链接库 ---------------------
EXTRA_LIBS  = -lbase -lm -lpthread
EXTRA_LIBPATH = -L$(TARGET_LIBS_DIR)/lib

# --------------------- 板级特定 CFLAGS ---------------------
PLAT_CFLAGS = -DPLAT_SSU9383

# --------------------- 交叉编译说明 ---------------------
# 交叉编译时会优先使用与当前 PLAT 对应的目标库目录，避免再把宿主机架构库误链接到交叉目标上。
# 若缺少目标架构的库文件，可将其放入 libs/<arch>/ 下，并将 LIBS_ARCH 改为对应名称。
```

然后进入到这个文件夹最高级目录，输入命令

```
./build.sh ssu9383
```

然后就会看到输出内容

```shell
root@YTF-NB-00090A:/work/hdl# ./build.sh ssu9383

========================================
  Building HDL: PLAT=ssu9383
========================================

========================================
  构建完成: PLAT=ssu9383
  编译器: /opt/sigmastar_tool/gcc-10.2.1-20210303-sigmastar-glibc-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc
  输出目录: build-ssu9383/
  依赖库目录: libs/aikaid
  动态库: libhdl_core.so
           libhdl_drivers.so
           libhdl_test.so
========================================
归档到 out/ssu9383/ ...
mkdir -p out/ssu9383/lib
mkdir -p out/ssu9383/include/hdl
mkdir -p out/ssu9383/include/base
cp -f build-ssu9383/lib/libhdl_core.so out/ssu9383/lib/
cp -f build-ssu9383/lib/libhdl_drivers.so out/ssu9383/lib/
cp -f build-ssu9383/lib/libhdl_test.so out/ssu9383/lib/
cp -f libs/aikaid/lib/libbase.so out/ssu9383/lib/
cp -f code/include/*.h out/ssu9383/include/hdl/
cp -f code/test/cmd_hdl_test.h out/ssu9383/include/hdl/
cp -f libs/aikaid/include/*.h out/ssu9383/include/base/
========================================
  归档完成: out/ssu9383/
  lib/          : libhdl_core.so, libhdl_drivers.so, libhdl_test.so, libbase.so
  include/hdl/  : HDL 头文件
  include/base/ : base库头文件
========================================
[OK] PLAT=ssu9383 构建完成
root@YTF-NB-00090A:/work/hdl#
```

生成的编译文件为

```shell
root@YTF-NB-00090A:/work/hdl# ls
Makefile  build-ssu9383  build-virt  build.sh  code  docs  libs  out  plats  third
root@YTF-NB-00090A:/work/hdl# cd build-ssu9383/
root@YTF-NB-00090A:/work/hdl/build-ssu9383# tree
.
├── 01_all_devices_test
├── 02_hdl_api_test_cli
├── lib
│   ├── libhdl_core.so
│   ├── libhdl_drivers.so
│   └── libhdl_test.so
```

###adb指令

- 查询adb设备

```shell
adb devices
```

- adb进入设备

```shell
adb shell
```

- 推送

```shell
adb push xxx
```

### 串口调试

使用USB连接星宸板子的adb调试口，然后使用adb将对应的文件推送到星辰平台

```shell
adb push C:\Users\wuyoujian\Desktop\hdl_out\02_hdl_api_test_cli /tmp/
adb push C:\Users\wuyoujian\Desktop\hdl_out\lib /tmp/
```

然后进入到设备

```shell
PS C:\Users\wuyoujian> adb shell
/ # export LD_LIBRARY_PATH=/tmp/lib
/ # /tmp/02_hdl_api_test_cli -i
/bin/sh: /tmp/02_hdl_api_test_cli: Permission denied
/ # cd /tmp/
tmp # ls
02_hdl_api_test_cli  lib
tmp # chmod 777 02_hdl_api_test_cli
tmp # /tmp/02_hdl_api_test_cli -i
[1970-01-01 01:30:58] [INFO] [02_hdl_api_test_cli.c:main:91] HDL plat init OK
[1970-01-01 01:30:58] [INFO] [02_hdl_api_test_cli.c:main:123] HDL API test CLI(1) start...
Mini Shell (msh) - Type 'help' for commands, 'exit' to quit
msh>
```

`HDL plat init OK` 说明框架已经成功打开并配置了 `/dev/ttyS4`，UART4 的设备节点、权限、驱动链路全部打通

- **有 USB 转 TTL**：接 UART4 的 TX、RX、GND 三根线到转接板，PC 串口助手开 115200-8N1，然后 CLI 里：

```shell
test uart set_cfg uart4 115200 0
test uart send uart4 hello_uart4 
```

PC 串口助手显示出 `hello_uart4` 就是 TX 方向通了。再从 PC 侧发一串数据，CLI 里

```shell
test uart available uart4
test uart recv uart4 255 #将十六进制数变成ASCLL码
```

显示出 PC 发的内容就是 RX 方向通了，测试完成。

- **没有 USB 转 TTL**：把 UART4 的 TX 和 RX 引脚短接，CLI 里一条命令：

```shell
test uart set_cfg uart4 115200 0
test uart loopback uart4 abcdef
```

输出里 `sent=6 recv=6` 且内容是 `abcdef` 就通过。

```
msh> test uart set_cfg uart4 115200 0
  [INFO] [uart4] baud=115200 flow=0
  [PASS]
msh> test uart loopback uart4 abcdef
  [INFO] [uart4] loopback sent=6 avail=6 recv=6
  [PASS]
msh>
```

