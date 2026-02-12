# Session 3 - Task Excercise
* Understand the use of the mutex

## 1) Activity Goal

[x]  Create the 7 task code

[X] Task 1 Heartbeat
[X] Task 2 Alive task
[X] Task 3 Queue Struct Send
[X] Task 4 Queue Struct Receive
[X] Task 5 and 6 Mutex reading a button
[X] Task 7 Error loggin for task 1-6

## 2) Materials & Setup

- **Tools/Software** - Editors: VS Code, Python 3.12, ESP32-IDF

Wiring/Safety: ESP32-IDF, LED, BUTTON, RESISTANCE OF 1KOHM AND 220 OHMS

## 3) Procedure 

We start task by task, analyzing, how to adapt it to the previous one.

## 4) Data, Tests & Evidence
We understand from the evidence obtained that in some cases we have to change the priority of tasks for it to work properly, and the mutex part, could execute both tasks.

## 5) Analysis
Is necessary to asssign the priorities of the tasks in order for it to work properly, and to have a better organization of the tasks

## 6) Code
The final code done is the following one:
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO    GPIO_NUM_2
#define BUTTON_GPIO GPIO_NUM_19  

static const char *TAG = "HOMEWORK";

typedef struct {
    int id;
    int value;
} data_t;

static QueueHandle_t data_queue = NULL;
static SemaphoreHandle_t btn_mutex = NULL;

// Arreglo para la Task 7 
static uint32_t task_counters[6] = {0};


static void heartbeat_task(void *pvParameters) {
    while (1) {
        task_counters[0]++; // Avisa que está viva
        gpio_set_level(LED_GPIO, 1); 
        vTaskDelay(pdMS_TO_TICKS(400)); 
        gpio_set_level(LED_GPIO, 0); 
        vTaskDelay(pdMS_TO_TICKS(300)); 
        gpio_set_level(LED_GPIO, 1); 
        vTaskDelay(pdMS_TO_TICKS(300)); 
        gpio_set_level(LED_GPIO, 0); 
        vTaskDelay(pdMS_TO_TICKS(250));
    }
}

static void alive_task(void *pvParameters)
{
    int n = 0;
    while (1) {
        task_counters[1]++;
        ESP_LOGI(TAG, "Alive: n=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

static void queue_send_task(void *pvParameters) {
    data_t msg;

    int counter = 0;
    while (1) {

        task_counters[2]++;
        msg.id = 3;
        msg.value = counter++;
        
        if (xQueueSend(data_queue, &msg, pdMS_TO_TICKS(100)) == pdPASS) {
            ESP_LOGI("QUEUE_SEND", "Enviado a cola: ID=%d, Val=%d", msg.id, msg.value);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}


static void queue_receive_task(void *pvParameters) {
    data_t received_msg;
    while (1) {
        task_counters[3]++;
        if (xQueueReceive(data_queue, &received_msg, portMAX_DELAY) == pdPASS) {
            ESP_LOGI("QUEUE_RECV", "Recibido de cola: ID=%d, Val=%d", 
                     received_msg.id, received_msg.value);
        }
    }
}


static void mutex_button_task(void *pvParameters) {
    int task_id = (int)pvParameters; 
    const char* name = (task_id == 4) ? "Tarea 5" : "Tarea 6";

    while (1) {
        task_counters[task_id]++;
        
       
        if (xSemaphoreTake(btn_mutex, portMAX_DELAY) == pdTRUE) {
            int level = gpio_get_level(BUTTON_GPIO);
            ESP_LOGI("MUTEX_READ", "[%s] Leyó botón: %s", name, level ? "Presionado" : "Suelto");
            
          
            vTaskDelay(pdMS_TO_TICKS(150)); 
            
            xSemaphoreGive(btn_mutex); 
        }
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}


static void monitor_task(void *pvParameters) {
    uint32_t last_counters[6] = {0};
    
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(5000)); 
        ESP_LOGI("MONITOR", "Verficicando tareas");

        for (int i = 0; i < 6; i++) {
            if (task_counters[i] == last_counters[i]) {
                ESP_LOGI("MONITOR", "ERROR: Tarea %d detenida", i + 1);
            } else {
                last_counters[i] = task_counters[i];
            }
        }
    }
}


void app_main(void) {
    
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_GPIO);

    
    data_queue = xQueueCreate(5, sizeof(data_t));
    btn_mutex = xSemaphoreCreateMutex();

    if (data_queue != NULL && btn_mutex != NULL) {

        xTaskCreate(heartbeat_task,     "T1",  2048, NULL,   1, NULL);
        xTaskCreate(alive_task,         "T2",  2048, NULL,   1, NULL);
        xTaskCreate(queue_send_task,    "T3",   2048, NULL,   2, NULL);
        xTaskCreate(queue_receive_task, "T4",   2048, NULL,   2, NULL);
        

        xTaskCreate(mutex_button_task,  "T5",    2048, (void*)4, 3, NULL);
        xTaskCreate(mutex_button_task,  "T6",    2048, (void*)5, 3, NULL);

        xTaskCreate(monitor_task,       "T7",    2048, NULL,   5, NULL);
        
    }
}
```

## 7 files and media
* This is the video of it working.
<iframe width="560" height="315" src="https://www.youtube.com/embed/qte915losIw" frameborder="0" allowfullscreen></iframe>