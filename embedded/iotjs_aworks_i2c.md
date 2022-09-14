---
title: IOT.js适配AWorks平台通用外设接口（3）：I2C
date: 2022-04-17
categories:
 - 嵌入式
tags:
 - C
 - 嵌入式
 - AWorks
 - IOT.js
---

## 1 前言

近期因工作需求学习了一下 IOT.js 和 AWorks 平台通用外设接口（包括：ADC、GPIO、I2C、PWM、SPI 和 UART），并将它们逐一适配到 IOT.js 中，为后续 [AWTK-MVMM](https://github.com/zlgopen/awtk-mvvm) 的 JS项目支持平台外设调用奠定基础，此处做笔记记录一下。

- [IOT.js适配AWorks平台通用外设接口（1）：ADC](./iotjs_aworks_adc.md)；
- [IOT.js适配AWorks平台通用外设接口（2）：GPIO](./iotjs_aworks_gpio.md)；
- [IOT.js适配AWorks平台通用外设接口（3）：I2C](./iotjs_aworks_i2c.md)；
- [IOT.js适配AWorks平台通用外设接口（4）：PWM](./iotjs_aworks_pwm.md)；
- [IOT.js适配AWorks平台通用外设接口（5）：SPI](./iotjs_aworks_spi.md)；
- [IOT.js适配AWorks平台通用外设接口（6）：UART](./iotjs_aworks_uart.md)；

> 备注：IOT.js 和 AWorks 的相关介绍请看第一篇 ADC 适配笔记。

## 2 I2C

### 2.1 I2C总线

I2C总线（Inter Integrated Circuit）是NXP公司开发的用于连接微控制器与外围器件的两线制总线，不仅硬件电路非常简洁，而且还具有极强的复用性和可移植性。I2C总线不仅适用于电路板内器件之间的通信，而且通过中继器还可以实现电路板与电路板之间长距离的信号传输。

### 2.2 I2C接口

绝大部分情况下，MUC都作为I2C主机与I2C从机器件通信，因此这里仅介绍AWorks中将MCU作为I2C主机的相关接口：

- aw_i2c_mkdev：初始化I2C从机实例。
- aw_i2c_read：I2C读操作。
- aw_i2c_write：I2C写操作。

## 3 适配过程

### 3.1 AWorks演示代码

先来看看I2C的基本读写功能，我们在底板上跑一下简单的例程。

步骤一：外设使能，在AWorks工程配置文件 `aw_prj_params.h` 中开启以下宏定义使能对应的I2C从机所需的设备：

```c
#define AW_DEV_IMX1050_LPI2C1 /**< \brief iMX1050 LPI2C1 (I2C1)*/
#define AW_DEV_EXTEND_PCF85063 /**< \brief PCF85063 实时时钟 (需要配套I2C外设) */
```

> 备注：如果没有对应的外设，可以暂时改用RTC0来测试，这个设备也使用 I2C 协议通信。RTC0设备地址 `0x51`，器件内部子地址（命令）`0x00`，表示读写设备内部的第一个字节。

步骤二：到外设文件中查看设备对应的引脚，比如这里查看 `awbl_hwconf_imx1050_lpi2c1.h` 文件，可以看到I2C设备使用的引脚有 `GPIO1_17` 和 `GPIO1_16`，我们需确定这两个引脚能正常使用。

步骤三：编写例程，读写I2C从机中的数据。示例代码如下：


```c
#include "aw_i2c.h"
#include "string.h"

#define I2C_SLAVE_ADDR      0x7EU      /* 从机地址(可查看具体I2C从机的配置文件，根据配置文件修改从机地址) */
#define I2C_MASTER_BUSID    DE_I2C_ID  /* 从机使用I2C1(可查看具体I2C从机的配置文件，根据配置文件修改I2C总线) */
#define SLAVE_REG_SEC       0x02       /* 器件内子地址，从此地址开始写入数据 */

int main()
{
    aw_i2c_device_t     i2c_slave;
    uint8_t             wr_data[10];
    uint8_t             rd_data[10];
    int                 i = 0;
    int                 ret;

    aw_kprintf("I2C_SYNC_Test:\r\n");

    /* 初始化设备实例，从机地址占7Bit，器件内部子地址占1字节 */
    aw_i2c_mkdev(&i2c_slave,
                 I2C_MASTER_BUSID,
                 I2C_SLAVE_ADDR,
                 AW_I2C_ADDR_7BIT | AW_I2C_SUBADDR_1BYTE);

    /* 清除缓存区 */
    memset(wr_data, 1, 10);
    memset(rd_data, 1, 10);

    for(i = 0; i < 10; i++) {
        wr_data[i] = i;
    }

    /* 写数据到从机设备 */
    ret = aw_i2c_write(&i2c_slave, SLAVE_REG_SEC, &wr_data[0], 10);
    if (ret != AW_OK) {
        aw_kprintf("aw_i2c_write fail:%d\r\n", ret);
        return ;
    } else {
        aw_kprintf("aw_i2c_write OK!\r\n");
    }

    /* 从从机设备中读取数据 */
    ret = aw_i2c_read(&i2c_slave, SLAVE_REG_SEC, &rd_data[0], 10);
    if (ret != AW_OK) {
        aw_kprintf("aw_i2c_read fail:%d\r\n", ret);
        return ;
    } else {
        aw_kprintf("aw_i2c_read OK!\r\n");
    }

    /* 数据校验 */
    ret = strcmp((const char * )wr_data, (const char *)rd_data);
    if (ret != AW_OK) {
        aw_kprintf("verify fail:%d\r\n", ret);
    } else {
        aw_kprintf("verify OK!\r\n");
    }

    return 0;
}
```

输出结果：

```bash
I2C_SYNC_Test:
aw_i2c_write OK!
aw_i2c_read OK!
verify OK!
```

### 3.2 C语言适配层

在 IOT.js 中，适配某个平台的外设通常需要实现 `src/modules/iotjs_module_xxx.h` 文件中的接口，比如这里我们需要实现 `iotjs_module_i2c.h` 中的相关接口：

```c
#ifndef IOTJS_MODULE_I2C_H
#define IOTJS_MODULE_I2C_H

#include "iotjs_def.h"
#include "iotjs_module_periph_common.h"

// Forward declaration of platform data. These are only used by platform code.
// Generic I2C module never dereferences platform data pointer.
typedef struct iotjs_i2c_platform_data_s iotjs_i2c_platform_data_t;
// This I2c class provides interfaces for I2C operation.
typedef struct {
  jerry_value_t jobject;
  iotjs_i2c_platform_data_t* platform_data;

  char* buf_data;
  uint8_t buf_len;
  uint8_t address;
} iotjs_i2c_t;

jerry_value_t iotjs_i2c_set_platform_config(iotjs_i2c_t* i2c,
                                            const jerry_value_t jconfig);
bool iotjs_i2c_open(iotjs_i2c_t* i2c);
bool iotjs_i2c_write(iotjs_i2c_t* i2c);
bool iotjs_i2c_read(iotjs_i2c_t* i2c);
bool iotjs_i2c_close(iotjs_i2c_t* i2c);

// Platform-related functions; they are implemented
// by platform code (i.e.: linux, nuttx, tizen).
void iotjs_i2c_create_platform_data(iotjs_i2c_t* i2c);
void iotjs_i2c_destroy_platform_data(iotjs_i2c_platform_data_t* platform_data);

#endif /* IOTJS_MODULE_I2C_H */
```

适配层（`src/modules/aworks/iotjs_module_i2c-aworks.c`）代码如下：

```c
#if !defined(WITH_AWORKS)
#error "Module __FILE__ is for AWorks only"
#endif

#include "iotjs_def.h"
#include "aw_i2c.h"
#include "modules/iotjs_module_i2c.h"

struct iotjs_i2c_platform_data_s {
  int bus;
  aw_i2c_device_t i2c_slave;
};

/* 默认从机地址占7Bit，器件子地址占1字节，数据包首字节为器件地址 */
#define AWORKS_I2C_DEV_FLAGS \
  (AW_I2C_ADDR_7BIT | AW_I2C_SUBADDR_1BYTE | AW_I2C_SUBADDR_NONE)

#define AWORKS_I2C_SLAVE_REG_SEC 0X00

void iotjs_i2c_create_platform_data(iotjs_i2c_t* i2c) {
  i2c->platform_data = IOTJS_ALLOC(iotjs_i2c_platform_data_t);
  i2c->platform_data->bus = -1;
}

void iotjs_i2c_destroy_platform_data(iotjs_i2c_platform_data_t* platform_data) {
  IOTJS_ASSERT(platform_data);
  IOTJS_RELEASE(platform_data);
}

jerry_value_t iotjs_i2c_set_platform_config(iotjs_i2c_t* i2c,
                                            const jerry_value_t jconfig) {
  iotjs_i2c_platform_data_t* platform_data = i2c->platform_data;

  JS_GET_REQUIRED_CONF_VALUE(jconfig, platform_data->bus,
                             IOTJS_MAGIC_STRING_BUS, number);

  return jerry_create_undefined();
}

bool iotjs_i2c_open(iotjs_i2c_t* i2c) {
  iotjs_i2c_platform_data_t* platform_data = i2c->platform_data;
  IOTJS_ASSERT(platform_data);

  aw_i2c_mkdev(&platform_data->i2c_slave, platform_data->bus, i2c->address,
               AWORKS_I2C_DEV_FLAGS);
  if (platform_data->i2c_slave.addr != i2c->address) {
    DLOG("%s: I2C init slave error", __func__);
    return false;
  }

  return true;
}

bool iotjs_i2c_close(iotjs_i2c_t* i2c) {
  iotjs_i2c_platform_data_t* platform_data = i2c->platform_data;
  IOTJS_ASSERT(platform_data);

  if (platform_data->i2c_slave.addr != i2c->address) {
    DLOG("%s: cannot close I2C", __func__);
  }

  platform_data->bus = -1;
  memset(&platform_data->i2c_slave, 0x00, sizeof(aw_i2c_device_t));

  return true;
}

bool iotjs_i2c_write(iotjs_i2c_t* i2c) {
  aw_err_t ret;
  uint8_t len = i2c->buf_len;
  uint8_t* data = (uint8_t*)i2c->buf_data;
  iotjs_i2c_platform_data_t* platform_data = i2c->platform_data;
  IOTJS_ASSERT(len > 0 && data != NULL);

  ret = aw_i2c_write(&platform_data->i2c_slave, AWORKS_I2C_SLAVE_REG_SEC, data,
                     len);
  IOTJS_RELEASE(i2c->buf_data);

  if (ret != AW_OK) {
    DLOG("%s: cannot write data(%d)", __func__, ret);
    return false;
  }

  return true;
}

bool iotjs_i2c_read(iotjs_i2c_t* i2c) {
  aw_err_t ret;
  uint8_t len = i2c->buf_len;
  IOTJS_ASSERT(len > 0);

  i2c->buf_data = iotjs_buffer_allocate(len);
  iotjs_i2c_platform_data_t* platform_data = i2c->platform_data;
  IOTJS_ASSERT(i2c->buf_data != NULL);

  ret = aw_i2c_read(&platform_data->i2c_slave, AWORKS_I2C_SLAVE_REG_SEC,
                    (uint8_t*)i2c->buf_data, len);

  if (ret != AW_OK) {
    DLOG("%s: cannot read data(%d)", __func__, ret);
    return false;
  }

  return true;
}
```

### 3.2 JS测试代码

适配好后，我们编写 JS 代码测试一下，这里直接采用 RTC0 设备进行测试，该设备也采用 I2C 协议与 MCU 通信：

```js
var i2c = require('i2c');  /* 导入i2c模块 */

var configuration = {
  address: 0x51,  /* 从机地址 */
  bus: 1          /* I2C总线编号 */
};

console.log('I2C sync function test');
var loopCnt = 5;
var i2cObj = i2c.openSync(configuration);
var loop = setInterval(function() {
  /* 第一字节为器件内部字地址，第二字节为数据 */
  i2cObj .writeSync([0x00, loopCnt]);
  var res = i2cObj .readSync(2);  /* 读取两个字节的数据 */
  console.log('read result:', res[1]);

  if (--loopCnt <= 0) {
    clearInterval(loop);
    i2cObj.closeSync();
  }
}, 800);
```

输出结果：

```bash
I2C sync function test
read result:5
read result:4
read result:3
read result:2
read result:1
```
