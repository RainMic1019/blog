---
title: 嵌入式Linux(awtk-linux-fb)双屏显示
date: 2021-09-05
categories:
 - 嵌入式
tags:
 - C
 - GUI
 - AWTK
 - 嵌入式
 - Linux
---

## 1 前言

近期尝试了在嵌入式 Linux 上适配双屏显示，即外接两个显示屏，同步显示 GUI 界面，其难点主要在于从 Flush 时将图像拷贝到两个 LCD 设备中，本文做个记录。

> 注意：本文基于 AWTK 针对 arm-linux 平台的移植适配双屏显示。
> 
> AWTK 是为嵌入式系统开发的 GUI 引擎库，GitHub 地址：[https://github.com/zlgopen/awtk](https://github.com/zlgopen/awtk)。
> awtk-linux-fb 是 AWTK 针对 arm-linux 平台的移植，GitHub 地址：[https://github.com/zlgopen/awtk-linux-fb](https://github.com/zlgopen/awtk-linux-fb)。

##  2 双屏显示的原理

awtk-linux-fb 基于 Framebuffer 实现的 LCD 支持两个屏幕显示比较简单，其原理是在 Flush 的时候将离线显存 offline 中的图像拷贝到两个显示屏的显存 online 中，需要注意的是，这会消耗性能，降低帧率。

了解以上原理后，可以明白，无论是双屏显示还是三屏显示，甚至多屏显示，都可以通过这种简单的软件拷贝方式实现，但 GUI 刷新效率肯定会严重降低，一般这种多屏显示都通过硬件完成，软件实现只提供一个实在没办法的解决方案。

> 备注：offline 是 AWTK 用于绘制的离线显存；online 是用于 LCD 屏幕显示的在线显存，即真正的系统显存。

##  3 如何实现双屏显示

### 3.1 确认两个屏幕的驱动文件

在嵌入式 Linux 平台中，两个屏幕通常会对应两个驱动文件，比如 /dev/fb0 和 /dev/fb1，需要注意的是，在 Flush 中直接拷贝图像要求这两个 LCD 设备的分辨率和颜色格式必须一致。当它们不一致时也可以在拷贝前做内部处理，但会进一步降低效率，因此不建议这样做。

### 3.2 初始化时打开LCD驱动文件

AWTK 初始化时，需要 lcd_t 对象，该对象用于关联硬件设备 LCD，实现显示功能，如需要双屏显示，那么此时就需要打开两个 LCD 驱动文件，并初始化相关信息，代码如下：

```c
/* awtk-linux-fb/awtk-port/lcd_linux/lcd_linux_fb.c */
/* 打开并初始化两个 LCD 设备 */
static bool_t lcd_linux_fb_open(fb_info_t* fb, fb_info_t* fb_ex, const char* filename, const char* filename_ex) {
  return_value_if_fail(fb !=  NULL && filename != NULL, FALSE);
  if (fb_open(fb, filename) == 0) {
    if (fb_ex != NULL && filename_ex != NULL){
      if (fb_open(fb_ex, filename_ex) != 0) {
        fb_close(fb);
        return FALSE;
      }
    }
    return TRUE;
  }
  return FALSE;
}

/* 创建 lcd_t 对象，此处需要两个 LCD 驱动文件名 */
lcd_t* lcd_linux_fb_create_ex(const char* filename, const char* filename_ex) {
  lcd_t* lcd = NULL;
  fb_info_t* fb = &s_fb;
  fb_info_t* fb_ex = &s_fb_ex;
  return_value_if_fail(filename != NULL && filename_ex != NULL, NULL);

  if (lcd_linux_fb_open(fb, fb_ex, filename, filename_ex)) {
    lcd = lcd_linux_create(fb, fb_ex);
  }

  atexit(on_app_exit);
  signal(SIGINT, on_signal_int);

  return lcd;
}
```

> 备注：fb_open 函数的实现详见：awtk-linux-fb/awtk-port/lcd_linux/fb_info.h。

### 3.3 重载lcd_t对象的flush函数

通常两个显示屏分别对应两块显存，而 AWTK 默认的 flush 函数只拷贝到一个显示屏的显存上，由于此处需要将 GUI 拷贝到两块显存，所以需要重载 lcd_t 对象的 flush 函数，代码如下：

```c
/* awtk-linux-fb/awtk-port/lcd_linux/lcd_linux_fb.c */
/* 创建基于 Flush 模式刷新屏幕的 lct_t 对象  */
static lcd_t* lcd_linux_create_flushable(fb_info_t* fb) {
  lcd_t* lcd = NULL;
  int w = fb_width(fb);
  int h = fb_height(fb);
  int line_length = fb_line_length(fb);

  uint8_t* online_fb = (uint8_t*)(fb->fbmem0);
  uint8_t* offline_fb = (uint8_t*)(fb->fbmem_offline);
  
  /* 根据 LCD 设备的 bpp 和颜色格式创建 lcd_t 对象 */
  lcd = lcd_mem_bgr565_create_double_fb(w, h, online_fb, offline_fb);

  if (lcd != NULL) {
    lcd->flush = lcd_mem_linux_flush;  /* 重载 flush 函数*/
    lcd_mem_set_line_length(lcd, line_length);
  }

  return lcd;
}
```

### 3.4 实现双屏图像拷贝

重载 lcd_t 对象的 flush 函数后，只需在重载的 lcd_mem_linux_flush 函数中，将 offline 图像拷贝到两个屏幕的显存（online）上即可，代码如下：

```c
/* awtk-linux-fb/awtk-port/lcd_linux/lcd_linux_fb.c */
/* 拷贝图像 */
static ret_t lcd_mem_linux_flush_ex(lcd_t* lcd, uint8_t* buff) {
  uint32_t rect_nr = 0;
  bitmap_t offline_fb;
  bitmap_t online_fb;
  bitmap_t online_fb_ex;
  fb_info_t* fb = &s_fb;
  fb_info_t* fb_ex = &s_fb_ex;
  uint8_t* buff_ex = fb_ex->fbmem0;
  const dirty_rects_t* dirty_rects;
  lcd_mem_t* mem = (lcd_mem_t*)lcd;
  system_info_t* info = system_info();
  lcd_orientation_t o = info->lcd_orientation;
  graphic_buffer_t *gb  = graphic_buffer_create_with_data(NULL, fb_width(fb_ex), fb_height(fb_ex), mem->format);
  return_value_if_fail(lcd != NULL && buff != NULL, RET_BAD_PARAMS);
  
  /* 绑定 offline 和两个 online 对应的位图 */
  lcd_linux_init_drawing_fb(mem, &offline_fb);
  lcd_linux_init_online_fb(mem, &online_fb, mem->online_gb, buff, fb_width(fb), fb_height(fb), fb_line_length(fb));
  lcd_linux_init_online_fb(mem, &online_fb_ex, gb, buff_ex, fb_width(fb_ex), fb_height(fb_ex), fb_line_length(fb_ex));

  dirty_rects = lcd_get_dirty_rects(lcd);  /* 获取脏矩形列表 */
  if (dirty_rects != NULL && dirty_rects->nr > 0 && gb != NULL) {
    uint32_t i = 0;
    rect_nr = dirty_rects->disable_multiple ? 1 : dirty_rects->nr;
    for (i = 0; i < rect_nr; i++) {
      const rect_t* dr = rect_nr > 1 ? (const rect_t*)dirty_rects->rects + i : (const rect_t*)&(dirty_rects->max);
      /* 拷贝图像，若屏幕旋转，则先旋转再拷贝 */
      if (o == LCD_ORIENTATION_0) {
        image_copy(&online_fb, &offline_fb, dr, dr->x, dr->y);
        image_copy(&online_fb_ex, &offline_fb, dr, dr->x, dr->y);
      } else {
        image_rotate(&online_fb, &offline_fb, dr, o);
        image_rotate(&online_fb_ex, &offline_fb, dr, o);
      }
    }
  }

  if (gb != NULL) {
    graphic_buffer_destroy(gb);
  }
  return RET_OK;
}

/* 重载的 flush 函数 */
static ret_t lcd_mem_linux_flush(lcd_t* lcd) {
  fb_info_t* fb = &s_fb;
  fb_info_t* fb_ex = &s_fb_ex;

/* 等待垂直同步 */
#if __FB_WAIT_VSYNC
  fb_sync(fb);
#endif

  if (fb_width(fb) == fb_width(fb_ex) && fb_height(fb) == fb_height(fb_ex) && fb_bpp(fb) == fb_bpp(fb_ex)){
    lcd_mem_linux_flush_ex(lcd, fb->fbmem0);
  }

  return RET_OK;
}
```
