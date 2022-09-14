---
title: IOT.js适配AWorks平台通用外设接口（6）：UART
date: 2022-05-02
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

## 2 UART

### 2.1 UART总线

UART（Universal Asynchronous Receiver/Transmitter）是一种通用异步收发传输器，其使用串行的方式在双机之间进行数据交换，实现全双工通信，数据引脚仅包含用于接收数据的RXD和用于发送数据的TXD。数据在数据线上按一位一位的串行传输，要正确解析这些数据，必须遵循UART协议，作为了解，这里仅简要讲述几个关键的概念：

- 波特率：它决定了数据传输的速率，其表示每秒传数据的位数，值越大，数据通信的速率越高，数据传输得越快。常见的波特率有：4800、9600、14400、19200、38400、115200等等，若波特率为115200，则表示每秒钟可以传输115200位（bit）数据。

- 空闲位：数据线上没有数据传输时，数据线处于空闲状态。空闲状态的电平逻辑为”1“。

- 起使位：它表示一帧数据传输的开始，起使位的电平逻辑是”0“。

- 数据包：紧接起始位后，即位实际通信传输的数据，数据的位数可以是5、6、7、8等，数据传输时，从最低位开始依次传输。

- 奇偶校验位：奇偶校验位用于接受方对数据进行校验，即使发现由于通信故障等问题造成的错误数据，它是可选的，可以不适用奇偶校验位。

- 停止位：它表示一帧数据的结束，其电平逻辑为”1“，其宽度可以是1位、1.5位、2位。即其持续的时间为位数乘以传输一位的时间（由波特率决定），例如波特率位115200，则传输一位的时间为1/115200秒，约为8.68us。若停止位的宽度为1.5位，则表示停止位持续时间位：1.5 * 8.68 us，约等于 13 us。

常见的帧格式位：1位起始位，8位数据位，无校验，1位停止位。由于起使位的宽度恒为1位，不会变化，而数据位，校验位和停止位都是可变的，因此，往往在描述串口通信协议时，都只是描述其波特率、数据位，校验位和停止位，不再单独说明起使位。

> 注意：通信双方必须使用完全相同的协议，包括波特率、起始位、数据位、停止位等。如果协议不一致，则通信数据会错乱，不能正常通信。再通信中，若出现乱码的情况，应该首先检查通信双方所使用的协议是否一致。

### 2.2 串行接口

在AWorks中，定义了通用的串行接口，可以使用串行接口操作UART，实现数据的收发，相关接口如下：

- aw_serial_ioctl：UART控制。
- aw_serial_write：发送数据。
- aw_serial_read：接收数据。

## 3 适配过程

### 3.1 AWorks演示代码

先来看看这些UART相关接口的基本用法，我们在底板上跑一下简单的例程。

步骤一：外设使能，在AWorks工程配置文件 `aw_prj_params.h` 中开启以下宏定义使能COM0串口：

```c
 #define AW_DEV_IMX1050_LPUART1 /**< \brief iMX1050 LPUART1 (COM0) */
```

步骤二：到外设文件中查看设备对应的引脚，比如这里查看 `awbl_hwconf_imx1050_lpuart1.h` 文件，可以看到该设备的使用的引脚为`GPIO1_12`和`GPIO1_13`引脚，在底板上分别对应RXD和TXD引脚，我们需确定这些引脚能正常使用。

步骤三：编写例程，测试COM0串口的读写功能。示例代码如下：

```c

#include "aworks.h"
#include "aw_delay.h"
#include "aw_serial.h"
#include "aw_ioctl.h"

int main() {
#define  TEST_SERIAL_NUM   COM0

    char    buf[32];
    int     len = 0;
    int     i   = 0;

    /* 串口初始化配置 波特率115200 */
    aw_serial_ioctl(TEST_SERIAL_NUM, SIO_BAUD_SET, (void *)115200);

    /* 串口参数 ：8个数据位 1个停止位，无奇偶校验 */
    aw_serial_ioctl(TEST_SERIAL_NUM, SIO_HW_OPTS_SET, (void *)(CS8 | CLOCAL | CREAD));

    /* 串口模式 ：查询模式 发送字符串 */
    aw_serial_ioctl(TEST_SERIAL_NUM, SIO_MODE_SET, (void *)SIO_MODE_POLL);
    aw_serial_poll_write(TEST_SERIAL_NUM, "Hello,Enter Serial Poll Mode:\r\n", 32);
    
    for(i = 0; i < 10; i++) {
        len = aw_serial_poll_read(TEST_SERIAL_NUM, buf, 10);
        if (len > 0) {
            aw_serial_poll_write(TEST_SERIAL_NUM, buf, len);
        }
        aw_serial_poll_write(TEST_SERIAL_NUM, "\r\n", len);
    }

    /*
     * 再次配置串口模式 ：中断模式
     *
     */
    // 清空串口输入输出缓冲区
    aw_serial_ioctl(TEST_SERIAL_NUM, AW_FIOFLUSH, NULL);
    // 设置串口读取超时时间
    aw_serial_ioctl(TEST_SERIAL_NUM, AW_TIOCRDTIMEOUT, (void *)10);
    // 设置为中断模式
    aw_serial_ioctl(TEST_SERIAL_NUM, SIO_MODE_SET, (void *)SIO_MODE_INT);

    // 发送数据
    aw_serial_write(TEST_SERIAL_NUM, "Hello,Enter Serial INT  Mode:\r\n", 32);
    for(i = 0; i < 10; i++) {
        // 读取数据
        len = aw_serial_read(TEST_SERIAL_NUM, buf, 10);
        if (len > 0) {
            aw_serial_write(TEST_SERIAL_NUM, buf, len);
        }
        aw_serial_write(TEST_SERIAL_NUM, "\r\n", len);
    }

    for (;;) {
        aw_mdelay(1000);
    }
    return 0;
}
```

### 3.2 C语言适配层

在 IOT.js 中，适配某个平台的外设通常需要实现 `src/modules/iotjs_module_xxx.h` 文件中的接口，比如这里我们需要实现 `iotjs_module_uart.h` 中的相关接口：

```c
#ifndef IOTJS_MODULE_UART_H
#define IOTJS_MODULE_UART_H

#include "iotjs_def.h"
#include "iotjs_module_periph_common.h"

#ifndef UART_WRITE_BUFFER_SIZE
#define UART_WRITE_BUFFER_SIZE 512
#endif

typedef struct iotjs_uart_platform_data_s iotjs_uart_platform_data_t;

typedef struct {
  int device_fd;
  unsigned baud_rate;
  uint8_t data_bits;
  iotjs_string_t buf_data;
  unsigned buf_len;
  char* buf;
  iotjs_uart_platform_data_t* platform_data;
} iotjs_uart_t;

void iotjs_uart_handle_close_cb(uv_handle_t* handle);
void iotjs_uart_register_read_cb(uv_poll_t* uart_poll_handle);

void iotjs_uart_create_platform_data(iotjs_uart_t* uart);
jerry_value_t iotjs_uart_set_platform_config(iotjs_uart_t* uart,
                                             const jerry_value_t jconfig);
void iotjs_uart_destroy_platform_data(iotjs_uart_platform_data_t* pdata);

bool iotjs_uart_open(uv_handle_t* uart_poll_handle);
bool iotjs_uart_write(uv_handle_t* uart_poll_handle);

#endif /* IOTJS_MODULE_UART_H */
```

适配层（`src/modules/aworks/iotjs_module_uart-aworks.c`）代码如下：

```c
#if !defined(WITH_AWORKS)
#error "Module __FILE__ is for AWorks only"
#endif

#include "uv.h"
#include "iotjs_def.h"
#include "iotjs_uv_handle.h"
#include "modules/iotjs_module_uart.h"
#include "aw_serial.h"
#include "aw_delay.h"
#include "aw_ioctl.h"

#ifndef AWORKS_UART_WRITE_BUFFER_SIZE
#define AWORKS_UART_WRITE_BUFFER_SIZE 127
#endif

struct iotjs_uart_platform_data_s {
  iotjs_string_t device;
  uint32_t wait_time;
  int32_t buf_len;
  uv_mutex_t mutex;
  char buf[UART_WRITE_BUFFER_SIZE];
};

static int device_to_constant(const char* device) {
  int ret = -1;

  if (!strcmp(device, "COM0")) {
    ret = COM0;
  } else if (!strcmp(device, "COM1")) {
    ret = COM1;
  } else if (!strcmp(device, "COM2")) {
    ret = COM2;
  } else if (!strcmp(device, "COM3")) {
    ret = COM3;
  } else if (!strcmp(device, "COM4")) {
    ret = COM4;
  } else if (!strcmp(device, "COM5")) {
    ret = COM5;
  } else if (!strcmp(device, "COM6")) {
    ret = COM6;
  } else if (!strcmp(device, "COM7")) {
    ret = COM7;
  } else if (!strcmp(device, "COM8")) {
    ret = COM8;
  } else if (!strcmp(device, "COM9")) {
    ret = COM9;
  }

  return ret;
}

static int databits_to_constant(uint8_t dataBits) {
  switch (dataBits) {
    case 8:
      return CS8;
    case 7:
      return CS7;
    case 6:
      return CS6;
    case 5:
      return CS5;
  }
  return -1;
}

void iotjs_uart_create_platform_data(iotjs_uart_t* uart) {
  uart->platform_data = IOTJS_ALLOC(iotjs_uart_platform_data_t);
}

void iotjs_uart_destroy_platform_data(iotjs_uart_platform_data_t* pdata) {
  IOTJS_ASSERT(pdata);

  uv_mutex_destroy(&pdata->mutex);
  iotjs_string_destroy(&pdata->device);
  IOTJS_RELEASE(pdata);
}

jerry_value_t iotjs_uart_set_platform_config(iotjs_uart_t* uart,
                                             const jerry_value_t jconfig) {
  JS_GET_REQUIRED_CONF_VALUE(jconfig, uart->platform_data->device,
                             IOTJS_MAGIC_STRING_DEVICE, string);

  return jerry_create_undefined();
}

static int uv_poll_uart_try_read(uv_poll_t* handle) {
  char buf[512];
  int32_t len = 0, buf_len = 0, buf_start = 0;
  iotjs_uart_t* uart = (iotjs_uart_t*)IOTJS_UV_HANDLE_EXTRA_DATA(handle);

  uv_mutex_lock(&uart->platform_data->mutex);
  buf_len = min(UART_WRITE_BUFFER_SIZE - uart->platform_data->buf_len, sizeof(buf));
  uv_mutex_unlock(&uart->platform_data->mutex);


  if (buf_len > 0) {
    len = aw_serial_read(uart->device_fd, buf, buf_len);
    
    if (len > 0) {
      uv_mutex_lock(&uart->platform_data->mutex);
      buf_start = uart->platform_data->buf_len;
      assert(buf_start + len < UART_WRITE_BUFFER_SIZE);
      memcpy(uart->platform_data->buf + buf_start, buf, len);
      uart->platform_data->buf_len += len;
      uv_mutex_unlock(&uart->platform_data->mutex);
    } else {
      aw_mdelay(uart->platform_data->wait_time);
    }
  } else {
    aw_mdelay(1);
  }

  if (len > 0) {
    return 0;
  } else {
    return 1;
  }
}

bool iotjs_uart_open(uv_handle_t* uart_poll_handle) {
  aw_err_t ret;
  struct aw_serial_timeout  timeout;
  uv_poll_t* uv_poll_handle = (uv_poll_t*)uart_poll_handle;
  iotjs_uart_t* uart =
      (iotjs_uart_t*)IOTJS_UV_HANDLE_EXTRA_DATA(uart_poll_handle);
  int com = device_to_constant(iotjs_string_data(&uart->platform_data->device));
  int data_bits = databits_to_constant(uart->data_bits);

  if (com < 0 || data_bits < 0) {
    DLOG("%s: serial port number error or data_bits error(%d)", __func__, com);
    return false;
  }

  /* 设置串口波特率 */
  ret = aw_serial_ioctl(com, SIO_BAUD_SET, (void*)uart->baud_rate);
  if (ret != AW_OK) {
    DLOG("%s: cannot set baud rate(%d)", __func__, ret);
    return false;
  }

  /* 设置数据位数 */
  ret = aw_serial_ioctl(com, SIO_HW_OPTS_SET, (void*)data_bits);
  if (ret != AW_OK) {
    DLOG("%s: cannot set data bits(%d)", __func__, ret);
    return false;
  }

  timeout.rd_timeout = aw_ms_to_ticks(100);             /* 读总超时为100ms */
  timeout.rd_interval_timeout = aw_ms_to_ticks(100);    /* 码间超时为100ms */
  ret = aw_serial_timeout_set(com, &timeout);
  if (ret != AW_OK) {
    DLOG("%s: cannot set timeout(%d)", __func__, ret);
    return false;
  }
  uart->device_fd = com;
  iotjs_uart_register_read_cb(uv_poll_handle);

  uart->platform_data->buf_len = 0;
  uv_mutex_init(&uart->platform_data->mutex);
  memset(uart->platform_data->buf, 0x0, sizeof(uart->platform_data->buf));
  uv_poll_set_try_poll_function(uv_poll_handle, uv_poll_uart_try_read);
  uart->platform_data->wait_time = AWORKS_UART_WRITE_BUFFER_SIZE / (uart->baud_rate / 10 / 1000 + 1) / 2; 

  return true;
}

bool iotjs_uart_write(uv_handle_t* uart_poll_handle) {
  int ret = 0;
  iotjs_uart_t* uart =
      (iotjs_uart_t*)IOTJS_UV_HANDLE_EXTRA_DATA(uart_poll_handle);
  int com = device_to_constant(iotjs_string_data(&uart->platform_data->device));
  const char* buf_data = iotjs_string_data(&uart->buf_data);

  DDDLOG("%s - data: %s", __func__, buf_data);

  ret = aw_serial_write(com, buf_data, uart->buf_len);

  if (ret < 0) {
    DLOG("%s: write data failed(%d)", __func__, ret);
    return false;
  }

  return true;
}

void iotjs_uart_handle_close_cb(uv_handle_t* uart_poll_handle) {
  iotjs_uart_t* uart =
      (iotjs_uart_t*)IOTJS_UV_HANDLE_EXTRA_DATA(uart_poll_handle);
  iotjs_uart_destroy_platform_data(uart->platform_data);
}

int uart_read(iotjs_uart_t* uart, void* dst, unsigned int n) {
  int32_t buf_len = 0;
  uv_mutex_lock(&uart->platform_data->mutex);
  if (uart->platform_data->buf_len > 0) {
    buf_len = min(n, uart->platform_data->buf_len);
    assert(buf_len >= 0);

    memcpy(dst, uart->platform_data->buf, buf_len);
    uart->platform_data->buf_len -= buf_len;
    if (uart->platform_data->buf_len > 0) {
      memcpy(uart->platform_data->buf, uart->platform_data->buf + buf_len, uart->platform_data->buf_len);
    }

    assert(uart->platform_data->buf_len >= 0);
  }
  uv_mutex_unlock(&uart->platform_data->mutex);
  return buf_len;
}

```


### 3.2 JS测试代码

适配好后，我们编写 JS 代码测试一下：

```js
var uart = require('uart');

var configuration = {
  device: 'COM2', /* 串口3使用GPIO1_22和GPIO1_23引脚，在底板上分别对应RX3和TX3引脚 */
  baudRate: 115200,
  dataBits: 8,
};

var read = 0;
var write = 0;
var serial = uart.open(configuration, function (err) {
  console.log('open done');

  serial.on('data', function (data) {
    console.log('read result: ' + data.toString());
    read = 1;

    if (read && write) {
      serial.close(function (err) {
        if (err) {
          console.log('Have an error: ' + err.message);
        }
        console.log('on read data callback close done');
      });
    }
  });

  serial.write('Hello there?\n\r', function (err) {
    console.log('write done');
    write = 1;

    if (read && write) {
      console.log('on read data callback close start');
      serial.close(function (err) {
        if (err) {
          console.log('Have an error: ' + err.message);
        }
        console.log('on write data callback close done');
      });
    }
  });
});
```

输出结果：

```bash
open done
write done
read result: Hello there?
on read data callback close done
```
