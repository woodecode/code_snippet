### 需求

实现FreeRTOS和lvgl的结合

在FreeRTOS中，在任务函数中调用lv_task_handler()和lv_tick_inc(1) lv_timer_handler()这些函数。具体来说，可以在一个低优先级的任务中调用lv_task_handler()函数，以处理GUI任务。而对于定时器的处理，可以在一个独立的高优先级任务中调用lv_tick_inc(1);lv_timer_handler()函数，以确保定时任务及时执行，并保证GUI任务不会受到影响。

另外，还需要注意的是，在调用lv_task_handler()函数之前，需要先初始化LVGL库和GUI任务。这可以通过调用lv_init()和lvgl_create_tasks()函数来实现。而在调用lv_tick_inc(1);lv_timer_handler()函数之前，需要先启动定时器。这可以通过调用lvgl_start_timer()函数来完成。

在下面的代码中，创建了一个低优先级的GUI任务（lvgl_task），以及一个高优先级的定时器任（timer_task）。GUI任务在等待信号量时会进入阻塞态，以避免资源的浪费。而定时器任务则每1ms更新一次定时器，以确保GUI任务的及时处理。另外，在app_main()函数中，我们先初始化LVGL库，再创建GUI任务和定时器任务，最后启动调度器。

### 代码
```c
/* FreeRTOS头文件 */
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "freertos/timers.h"

/* LVGL头文件 */
#include "lvgl/lvgl.h"
#include "lvgl_helpers.h"

/* 定义任务句柄 */
TaskHandle_t lvgl_task_handle = NULL;
/* 定义信号量 */
SemaphoreHandle_t lvgl_sem;
/* 定义定时器句柄 */
TimerHandle_t lvgl_timer_handle = NULL;

/* 定义定时器回调函数 */
void lvgl_timer_callback(TimerHandle_t xTimer)
{
    /* 更新系统时钟和定时器 */
    lv_tick_inc(1);
    lv_timer_handler();

    /* 发送信号量，唤醒GUI任务 */
    xSemaphoreGive(lvgl_sem);
}

/* 定义任务函数 */
void lvgl_task(void *pvParameters)
{
    while (1)
    {
        /* 等待信号量 */
        xSemaphoreTake(lvgl_sem, portMAX_DELAY);
        /* 处理LVGL任务 */
        lv_task_handler();
    }
}

void app_main()
{
    /* 初始化LVGL库 */
    lv_init();
    /* 创建GUI任务 */
    lvgl_create_tasks();
    /* 创建信号量 */
    lvgl_sem = xSemaphoreCreateBinary();
    /* 启动调度器后，创建GUI任务 */
    xTaskCreate(lvgl_task, "lvgl_task", 2048, NULL, 2, &lvgl_task_handle);
    /* 创建定时器 */
    lvgl_timer_handle = xTimerCreate("lvgl_timer", pdMS_TO_TICKS(1), pdTRUE, NULL, lvgl_timer_callback);
    /* 启动定时器 */
    xTimerStart(lvgl_timer_handle, 0);
    /* 启动调度器 */
    vTaskStartScheduler();
}

```

