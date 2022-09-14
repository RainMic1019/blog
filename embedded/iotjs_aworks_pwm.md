---
title: IOT.js适配AWorks平台通用外设接口（4）：PWM
date: 2022-04-18
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

## 2 PWM

### 2.1 PWM 简介

大小和方向随时间发生周期性变化的电流称为交流电，交流中最基本的波形称为正弦波，除此之外的波形称为非正弦波。计算机、电视机、雷达灯装置中使用的信号称为脉冲波，锯齿波等，其电压和电流波形都是非正弦交流的一种。

PWM（Pulse Width Modulation）就是脉冲宽度调制的意思，一种脉冲编码技术，即可以按照信号电平改变脉冲宽度。而脉冲宽度调制波的周期也是固定的，用占空比（高电平/周期，有效电平在整个信号周期中的时间比率，范围0~100%）来表示编码数值。PWM可以用于模拟信号电平进行数字编码，也可以通过高电平（或低电平）在整个周期中的时间来控制输出的能量，从而控制电机转速或LED亮度、蜂鸣器响度等。

简单理解，PWM 是按一定的规则对各脉冲的宽度进行调制，既可以改变逆变电路输出电压的大小，也可以改变输出频率。在下本的例程中体现在控制无源蜂鸣器的响度和频率。

### 2.2 PWM 接口

AWorks 提供了用于控制 PWM 的接口函数：

- aw_pwm_config：配置PWM（周期时间、脉宽时间等）；
- aw_pwm_enable：使能PWM输出；
- aw_pwm_disable：禁能PWM输出。

## 3 适配过程

### 3.1 AWorks演示代码

先来看看这些PWM相关接口的基本用法，我们在底板上跑一下简单的例程。

步骤一：外设使能，在AWorks工程配置文件 `aw_prj_params.h` 中开启以下宏定义使能蜂鸣器设备以及相关的计时器：

```c
#define AW_DEV_PWM_BUZZER          /**< \brief PWM Buzzer(蜂鸣器，需要配套 PWM) */
#define AW_DEV_IMX1050_QTIMER3_PWM /**< \brief iMX1050 QTimer3 PWM (与 PWM Buzzer 配套的 PWM，需要开启) */
```

步骤二：到外设文件中查看设备对应的引脚，比如这里查看 `awbl_hwconf_imx1050_qtimer3_pwm.h` 文件，可以看到该设备使用的引脚为 `GPIO1_18` ，我们需确定这个引脚能正常使用。

步骤三：编写例程，设置蜂鸣器的周期与脉宽占比。示例代码如下：

```c
#include "aw_pwm.h"

#define AW_PWM_ID   DE_BUZZER_PWMID /* 蜂鸣器PWM ID */

int main()
{
    uint32_t    period1 = 2000000;      /* (ns) */
    uint32_t    period2 = 1000000;      /* (ns) */

    aw_kprintf("\nPWM demo testing...\n");

    while(1) {

        /* 配置 PWM 的有效时间（高电平时间）50% ,周期 period1*/
        aw_pwm_config(AW_PWM_ID, period1 / 2, period1);
        aw_pwm_enable(AW_PWM_ID);      /* 使能通道 */
        aw_mdelay(250);
        aw_pwm_disable(AW_PWM_ID);     /* 禁能通道 */
        aw_mdelay(250);

        /* 配置 PWM 的有效时间（高电平时间）2% ,周期 period1*/
        aw_pwm_config(AW_PWM_ID, period1 / 50, period1);
        aw_pwm_enable(AW_PWM_ID);      /* 使能通道 */
        aw_mdelay(250);
        aw_pwm_disable(AW_PWM_ID);     /* 禁能通道 */
        aw_mdelay(250);

        /* 配置 PWM 的有效时间（高电平时间）50% ,周期 period2*/
        aw_pwm_config(AW_PWM_ID, period2 / 2, period2);
        aw_pwm_enable(AW_PWM_ID);      /* 使能通道 */
        aw_mdelay(250);
        aw_pwm_disable(AW_PWM_ID);     /* 禁能通道 */
        aw_mdelay(250);

        /* 配置 PWM 的有效时间（高电平时间）2% ,周期 period2*/
        aw_pwm_config(AW_PWM_ID, period2 / 50, period2);
        aw_pwm_enable(AW_PWM_ID);      /* 使能通道 */
        aw_mdelay(250);
        aw_pwm_disable(AW_PWM_ID);     /* 禁能通道 */
        aw_mdelay(250);
    }
    return 0;
}
```

这里的测试效果可以用示波器抓取波形来观察。

### 3.2 C语言适配层

在 IOT.js 中，适配某个平台的外设通常需要实现 `src/modules/iotjs_module_xxx.h` 文件中的接口，比如这里我们需要实现 `iotjs_module_pwm.h` 中的相关接口：

```c
#ifndef IOTJS_MODULE_PWM_H
#define IOTJS_MODULE_PWM_H

#include "iotjs_def.h"
#include "iotjs_module_periph_common.h"

#if defined(__TIZENRT__)
#include <iotbus_pwm.h>
#include <stdint.h>
#endif

// Forward declaration of platform data. These are only used by platform code.
// Generic PWM module never dereferences platform data pointer.
typedef struct iotjs_pwm_platform_data_s iotjs_pwm_platform_data_t;

typedef struct {
  jerry_value_t jobject;
  iotjs_pwm_platform_data_t* platform_data;

  uint32_t pin;
  double duty_cycle;
  double period;
  bool enable;
} iotjs_pwm_t;

jerry_value_t iotjs_pwm_set_platform_config(iotjs_pwm_t* pwm,
                                            const jerry_value_t jconfig);
bool iotjs_pwm_open(iotjs_pwm_t* pwm);
bool iotjs_pwm_set_period(iotjs_pwm_t* pwm);
bool iotjs_pwm_set_dutycycle(iotjs_pwm_t* pwm);
bool iotjs_pwm_set_enable(iotjs_pwm_t* pwm);
bool iotjs_pwm_close(iotjs_pwm_t* pwm);

// Platform-related functions; they are implemented
// by platform code (i.e.: linux, nuttx, tizen).
void iotjs_pwm_create_platform_data(iotjs_pwm_t* pwm);
void iotjs_pwm_destroy_platform_data(iotjs_pwm_platform_data_t* platform_data);

#endif /* IOTJS_MODULE_PWM_H */
```

适配层（`src/modules/aworks/iotjs_module_pwm-aworks.c`）代码如下：

```c
#if !defined(WITH_AWORKS)
#error "Module __FILE__ is for AWorks only"
#endif

#include "iotjs_def.h"
#include "aw_gpio.h"
#include "aw_pwm.h"
#include "modules/iotjs_module_pwm.h"

struct iotjs_pwm_platform_data_s {
  int pid;
};

/* PWM通道的id号 */
#define AWORKS_PWM_STRING_PID "pid"

#define AWORKS_GPIO_REQ_NAME "iotjs_pwm"

void iotjs_pwm_create_platform_data(iotjs_pwm_t* pwm) {
  pwm->platform_data = IOTJS_ALLOC(iotjs_pwm_platform_data_t);
  pwm->platform_data->pid = -1;
}

void iotjs_pwm_destroy_platform_data(iotjs_pwm_platform_data_t* platform_data) {
  IOTJS_RELEASE(platform_data);
}

jerry_value_t iotjs_pwm_set_platform_config(iotjs_pwm_t* pwm,
                                            const jerry_value_t jconfig) {
  JS_GET_REQUIRED_CONF_VALUE(jconfig, pwm->platform_data->pid,
                             AWORKS_PWM_STRING_PID, number);

  return jerry_create_undefined();
}

static bool iotjs_pwm_set_period_and_dutycycle(iotjs_pwm_t* pwm) {
  aw_err_t ret;
  unsigned long duty_ns;
  unsigned long period_ns;
  iotjs_pwm_platform_data_t* platform_data = pwm->platform_data;

  period_ns = (unsigned long)(1000 * 1000 * 1000 * pwm->period);
  duty_ns = (unsigned long)period_ns - (period_ns * pwm->duty_cycle);

  ret = aw_pwm_config(platform_data->pid, duty_ns, period_ns);

  return ret == AW_OK ? true : false;
}

bool iotjs_pwm_open(iotjs_pwm_t* pwm) {
  aw_err_t ret;
  int p_pins[] = { pwm->pin };

  ret = aw_gpio_pin_request(AWORKS_GPIO_REQ_NAME, p_pins, AW_NELEMENTS(p_pins));

  if (ret == AW_ERROR) {
    return iotjs_pwm_set_period_and_dutycycle(pwm);
  } else if (ret == -AW_ENXIO) {
    DLOG("%s: pwm pin number error(%d)", __func__, ret);
  } else if (ret == AW_OK) {
    aw_gpio_pin_release(p_pins, AW_NELEMENTS(p_pins));
    DLOG("%s: pwm pin is free, please check(%d)", __func__, ret);
  } else {
    DLOG("%s: pwm pin request unknow return(%d)", __func__, ret);
  }

  return false;
}

bool iotjs_pwm_set_period(iotjs_pwm_t* pwm) {
  return iotjs_pwm_set_period_and_dutycycle(pwm);
}

bool iotjs_pwm_set_dutycycle(iotjs_pwm_t* pwm) {
  return iotjs_pwm_set_period_and_dutycycle(pwm);
}

bool iotjs_pwm_set_enable(iotjs_pwm_t* pwm) {
  aw_err_t ret;
  iotjs_pwm_platform_data_t* platform_data = pwm->platform_data;

  if (pwm->enable) {
    ret = aw_pwm_enable(platform_data->pid);
  } else {
    ret = aw_pwm_disable(platform_data->pid);
  }

  return ret == AW_OK ? true : false;
}

bool iotjs_pwm_close(iotjs_pwm_t* pwm) {
  pwm->platform_data->pid = -1;
  return true;
}
```


### 3.2 JS测试代码

适配好后，我们编写 JS 代码测试一下，这里同样借助蜂鸣器进行测试，可以用示波器抓取波形查看效果：

```js
var pwm = require('pwm');  /* 导入pwm模块 */

var dutyCycles = [0.25, 0.5, 0.75];  /* 高低平占比 */
var frequencies = [1, 10, 30];       /* 频率，单位:kHz */

var configuration = {
  period: 0.001,  // 周期，单位：秒，表示1kHz
  dutyCycle: dutyCycles[0],
  pin: 18  // 引脚
  pid: 0   // PWM ID
};

function initPwm(pwm) {
  pwm.setPeriodSync(0.001);
  pwm.setDutyCycleSync(0.5);
}

var pwm0 = pwm.openSync(configuration);
console.log('PWM initialized');

pwm0.setEnableSync(true);
dutyCycleTest();

function dutyCycleTest() {
  var loopCnt = 0;

  var loop = setInterval(function() {
    if (pwm0 === null) {
      return;
    }

    if (loopCnt >= dutyCycles.length) {
      clearInterval(loop);
      initPwm(pwm0);
      console.log('PWM duty-cycle test complete');
      frequencyTest();
      return;
    }
    console.log("dutycycle(%d)", dutyCycles[loopCnt]);
    pwm0.setDutyCycleSync(dutyCycles[loopCnt++]);
  }, 1000);
}

function frequencyTest() {
  var loopCnt = 0;

  var loop = setInterval(function() {
    if (loopCnt >= frequencies.length) {
      clearInterval(loop);
      pwm0.setEnableSync(false);
      pwm0.closeSync();
      console.log('PWM frequency test complete');
      return;
    }
    console.log("frequency(%d)", frequencies[loopCnt]);
    pwm0.setFrequencySync(frequencies[loopCnt++]);
  }, 2000);
}
```

输出结果：

```bash
PWM initialized
dutycycle(0.25)
dutycycle(0.5)
dutycycle(0.75)
PWM duty-cycle test complete
frequency(1)
frequency(10)
frequency(30)
PWM frequency test complete
```
