# Session 3 - Lab Homework 

## 1) Exercise Goals

[x]  Read and answer the questions from Labs 1 trough 3

## 2) Materials & Setup

- **Tools/Software** - Editors: VS Code, Python 3.12
- **Hardware** - ESP32-C6, LED

## 3) Procedure 

## Lab 1
### First Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2   // CHANGE for your board

static const char *TAG = "LAB1";

static void blink_task(void *pvParameters)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(300));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

static void hello_task(void *pvParameters)
{
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "hello_task says hi, n=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 1 (two tasks)");

    // Stack size in ESP-IDF FreeRTOS is in BYTES
    xTaskCreate(blink_task, "blink_task", 2048, NULL, 5, NULL);
    xTaskCreate(hello_task, "hello_task", 2048, NULL, 5, NULL);
}
```
### Video of it working without modifications
<iframe width="560" height="315" src="https://www.youtube.com/embed/JBFUYanOvcM" frameborder="0" allowfullscreen></iframe>

### Experiment 1
* Priority experiment: change hello_task priority from 5 to 2.
#### New Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2   // CHANGE for your board

static const char *TAG = "LAB1";

static void blink_task(void *pvParameters)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(300));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

static void hello_task(void *pvParameters)
{
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "hello_task says hi, n=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 1 (two tasks)");

    // Stack size in ESP-IDF FreeRTOS is in BYTES
    xTaskCreate(blink_task, "blink_task", 2048, NULL, 5, NULL);
    xTaskCreate(hello_task, "hello_task", 2048, NULL, 2, NULL);
}
```
#### Video of it working 
<iframe width="560" height="315" src="https://youtube.com/shorts/dxw3BCAVOWQ" frameborder="0" allowfullscreen></iframe>

- **Does behavior change? Why might it (or might it not)?** - There is no noticeable change, due to the VTaskDelay() function, which makes that must of the time the tasks are in a Blocked state, and spend 99% of their time sleeping, so the low priority task could do it without a noticeable delay, it will matters if both tasks are ready to run at the same microsecond.



### Experiment  2
* Starvation demo: temporarily remove vTaskDelay(...) from hello_task.
#### New Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2   // CHANGE for your board

static const char *TAG = "LAB1";

static void blink_task(void *pvParameters)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(300));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

static void hello_task(void *pvParameters)
{
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "hello_task says hi, n=%d", n++);
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 1 (two tasks)");

    // Stack size in ESP-IDF FreeRTOS is in BYTES
    xTaskCreate(blink_task, "blink_task", 2048, NULL, 5, NULL);
    xTaskCreate(hello_task, "hello_task", 2048, NULL, 2, NULL);
}
```
#### Video of it working 
<iframe width="560" height="315" src="https://www.youtube.com/embed/-a6wAHV17UE" frameborder="0" allowfullscreen></iframe>

#### Video of it working with hello_task as a higher priority
<iframe width="560" height="315" src="https://www.youtube.com/embed/2-AvU6tteJM" frameborder="0" allowfullscreen></iframe>

- **What happens to blinking?** - The LED is still blinking, even if the priority is lower or the same as the hello_task, however, the however_task will run without delays in the moniitor, which in some cases will crash. In addition, if we change the priority of the hello_task to a higher one, the led will blink faster, making it loooks like it is completly on, because it is stuck in the Ready progress, it is showed in the second video.

- **Put the delay back and explain in one sentence why blocking helps.** - It helps, to move tasks to a blocked state and let other tasks to run until its time has passed.

## Lab 2 Queue: producer/consumer
### First Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"

static const char *TAG = "LAB2";
static QueueHandle_t q_numbers;

static void producer_task(void *pvParameters)
{
    int value = 0;

    while (1) {
        value++;

        // Send to queue; wait up to 50ms if full
        if (xQueueSend(q_numbers, &value, pdMS_TO_TICKS(50)) == pdPASS) {
            ESP_LOGI(TAG, "Produced %d", value);
        } else {
            ESP_LOGW(TAG, "Queue full, dropped %d", value);
        }

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

static void consumer_task(void *pvParameters)
{
    int rx = 0;

    while (1) {
        // Wait up to 1000ms for data
        if (xQueueReceive(q_numbers, &rx, pdMS_TO_TICKS(1000)) == pdPASS) {
            ESP_LOGI(TAG, "Consumed %d", rx);
        } else {
            ESP_LOGW(TAG, "No data in 1s");
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 2 (queue)");

    q_numbers = xQueueCreate(5, sizeof(int)); // length 5
    if (q_numbers == NULL) {
        ESP_LOGE(TAG, "Queue create failed");
        return;
    }

    xTaskCreate(producer_task, "producer_task", 2048, NULL, 5, NULL);
    xTaskCreate(consumer_task, "consumer_task", 2048, NULL, 5, NULL);
}
```
### Video of it working without modifications
<iframe width="560" height="315" src="https://www.youtube.com/embed/_gC4YUlhtd4" frameborder="0" allowfullscreen></iframe>

### Experiment 1
* Make the producer faster: change producer delay 200ms → 20ms.
#### New Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"

static const char *TAG = "LAB2";
static QueueHandle_t q_numbers;

static void producer_task(void *pvParameters)
{
    int value = 0;

    while (1) {
        value++;

        // Send to queue; wait up to 50ms if full
        if (xQueueSend(q_numbers, &value, pdMS_TO_TICKS(50)) == pdPASS) {
            ESP_LOGI(TAG, "Produced %d", value);
        } else {
            ESP_LOGW(TAG, "Queue full, dropped %d", value);
        }

        vTaskDelay(pdMS_TO_TICKS(20));
    }
}

static void consumer_task(void *pvParameters)
{
    int rx = 0;

    while (1) {
        // Wait up to 1000ms for data
        if (xQueueReceive(q_numbers, &rx, pdMS_TO_TICKS(1000)) == pdPASS) {
            ESP_LOGI(TAG, "Consumed %d", rx);
        } else {
            ESP_LOGW(TAG, "No data in 1s");
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 2 (queue)");

    q_numbers = xQueueCreate(5, sizeof(int)); // length 5
    if (q_numbers == NULL) {
        ESP_LOGE(TAG, "Queue create failed");
        return;
    }

    xTaskCreate(producer_task, "producer_task", 2048, NULL, 5, NULL);
    xTaskCreate(consumer_task, "consumer_task", 2048, NULL, 5, NULL);
}
```
#### Video of it working 
<iframe width="560" height="315" src="https://www.youtube.com/embed/skTfCmXJOWk" frameborder="0" allowfullscreen></iframe>

- **When do you see “Queue full”?** - I never see the queue full, since my consumer is faster than the consumer,because, it is awake like all the time, since the producer is sending information every 20 ms. But i should see it when the producer is putting items into the queue faster than the consumer can take them out.

### Experiment 2
* Increase the queue length 5 → 20.
#### Line of code changed
```bash
q_numbers = xQueueCreate(20, sizeof(int)); // length 5
```
#### Video of it modified 
<iframe width="560" height="315" src="https://www.youtube.com/embed/JBFUYanOvcM" frameborder="0" allowfullscreen></iframe>

- **What changed** - In my case there is no noticeable difference, because the consumer is faster than the consumer, because it has no delay and is awake every time the producer is sending data, so, is always ready to recieve the new message, making the queue always empty for new data.

### Experiment 3
* Make the consumer “slow”: after a successful receive, add: vTaskDelay(pdMS_TO_TICKS(300));

#### Video of it working
<iframe width="560" height="315" src="https://www.youtube.com/embed/-6PK8rryN7M" frameborder="0" allowfullscreen></iframe>

- **What pattern is happening now (buffering / backlog)?** - Both of them, since, for backlog my producer is faster than the consumer making a backlog of 20 numbers waiting to be processed, making also an overflow which is loosing data, the queue is full, due to the producer which is faster, so in some point it starts buffering.

## Lab 3 Mutex: protect a shared resource
### First Code Race Demo no mutex
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG = "LAB3A";

static volatile int shared_counter = 0;

static void increment_task(void *pvParameters)
{
    const char *name = (const char *)pvParameters;

    while (1) {
        // NOT safe: read-modify-write without protection
        int local = shared_counter;
        local++;
        shared_counter = local;

        if ((shared_counter % 1000) == 0) {
            ESP_LOGI(TAG, "%s sees counter=%d", name, shared_counter);
        }

        vTaskDelay(pdMS_TO_TICKS(1));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 3A (race demo)");

    xTaskCreate(increment_task, "incA", 2048, "TaskA", 5, NULL);
    xTaskCreate(increment_task, "incB", 2048, "TaskB", 5, NULL);
}
```
- **Why can the counter be wrong?** - Because, the task is divided in 3 instructions, one thhat reads, the other that adds and the last one that returns the value, and if any of the instructions is interrupted, it will loose an increment. 


### First code fix with a mutex
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "esp_log.h"

static const char *TAG = "LAB3B";

static volatile int shared_counter = 0;
static SemaphoreHandle_t counter_mutex;

static void increment_task(void *pvParameters)
{
    const char *name = (const char *)pvParameters;

    while (1) {
        xSemaphoreTake(counter_mutex, portMAX_DELAY);

        int local = shared_counter;
        local++;
        shared_counter = local;

        xSemaphoreGive(counter_mutex);

        if ((shared_counter % 1000) == 0) {
            ESP_LOGI(TAG, "%s sees counter=%d", name, shared_counter);
        }

        vTaskDelay(pdMS_TO_TICKS(1));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 3B (mutex fix)");

    counter_mutex = xSemaphoreCreateMutex();
    if (counter_mutex == NULL) {
        ESP_LOGE(TAG, "Mutex create failed");
        return;
    }

    xTaskCreate(increment_task, "incA", 2048, "TaskA", 5, NULL);
    xTaskCreate(increment_task, "incB", 2048, "TaskB", 5, NULL);
}
```
### Video of it working w/o modifications
<iframe width="560" height="315" src="https://www.youtube.com/embed/-6PK8rryN7M" frameborder="0" allowfullscreen></iframe>

- **Remove the mutex again. Do you ever see weird behavior?** - I don´t see a weird behaviour, compared to the other one.

#### Video of it working 
<iframe width="560" height="315" src="https://www.youtube.com/embed/3fsnEzzYnfg" frameborder="0" allowfullscreen></iframe>

## Change priorities: TaskA priority 6, TaskB priority 4.

- **What do you expect and why** - I expect that task A dominates the execution, and for it I could barely see task B

- **In one sentence: what does a mutex “guarantee”?** - It guarantees that only one task at a time can access a shared resource.