# MQTT LAB pt2

## 1) Exercise Goals
Connect 2 ESPS with MQTT to control on state and brightness

## 2) Materials & Setup
- **Tools/Software** - Editors: VS Code, Python 3.12, ESP32-C6

**Wiring/Safety:** - ESP32-C6 , LED, BUTTON, RESISTANCE OF  220 OHMS and 1KOHMS

## 3) Procedure 
The ESP32 works like a mini‑website that listens to commands and controls the LED, while making sure the responses are easy for apps or browsers to understand.

## Codes
### Main Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "nvs_flash.h"
#include "esp_log.h"

// Tus cabeceras personalizadas
#include "wifi_sta.h"
#include "mqtt_client_app.h"

static const char *TAG = "MAIN_APP";

// Credenciales y Broker
#define MI_SSID "iPhone de Sumie"
#define MI_PASS "12345678"
#define MI_BROKER "mqtt://test.mosquitto.org:1883"

void app_main(void) {
    // 1. Inicializar NVS (indispensable para Wi-Fi)
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 2. Conectar Wi-Fi usando tu función de laboratorio
    ESP_LOGI(TAG, "Iniciando Wi-Fi...");
    esp_err_t wifi_res = wifi_sta_connect(MI_SSID, MI_PASS, 30000);

    if (wifi_res == ESP_OK) {
        ESP_LOGI(TAG, "Wi-Fi Conectado. Iniciando MQTT...");
        
        // 3. Arrancar MQTT
        mqtt_app_start(MI_BROKER);
    } else {
        ESP_LOGE(TAG, "Error crítico: No se pudo conectar al Wi-Fi.");
    }

    // Heartbeat para saber que el sistema no se ha colgado
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(10000));
        ESP_LOGI(TAG, "Sistema activo...");
    }
}
```
### MQTT CLIENT CODE
```bash
#include <string.h>
#include <stdlib.h>
#include "esp_log.h"
#include "mqtt_client.h"
#include "driver/ledc.h" 
#include "driver/gpio.h" //  para los pines de dirección
#include "mqtt_client_app.h"

static const char *TAG = "MOTOR_DIR_LAB";

// Configuración Tópicos 
#define TOPIC_CMD   "ibero/ei2/team3/c6_01/cmd"
#define TOPIC_STAT  "ibero/ei2/team3/c6_01/status"

// Pines del Puente H 
#define PIN_AIN1       4   // Pin de dirección 1
#define PIN_AIN2       5   // Pin de dirección 2
#define PIN_PWM        8    // Pin de velocidad (PWM)

// --- Configuración PWM ---
#define LEDC_MODE      LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL   LEDC_CHANNEL_0
#define LEDC_TIMER     LEDC_TIMER_0
#define LEDC_RES       LEDC_TIMER_8_BIT

static esp_mqtt_client_handle_t client = NULL;

void init_motor_hardware() {
    // 1. Configurar pines de dirección como salida simple
    gpio_reset_pin(PIN_AIN1);
    gpio_set_direction(PIN_AIN1, GPIO_MODE_OUTPUT);
    gpio_reset_pin(PIN_AIN2);
    gpio_set_direction(PIN_AIN2, GPIO_MODE_OUTPUT);

    // 2. Configurar PWM para el pin de velocidad
    ledc_timer_config_t timer_cfg = {
        .speed_mode = LEDC_MODE,
        .timer_num = LEDC_TIMER,
        .duty_resolution = LEDC_RES,
        .freq_hz = 5000,
        .clk_cfg = LEDC_AUTO_CLK
    };
    ledc_timer_config(&timer_cfg);

    ledc_channel_config_t chan_cfg = {
        .speed_mode = LEDC_MODE,
        .channel = LEDC_CHANNEL,
        .timer_sel = LEDC_TIMER,
        .gpio_num = PIN_PWM,
        .duty = 0
    };
    ledc_channel_config(&chan_cfg);
}

void set_motor_speed(int speed) {
    if (speed > 0) {
        // Hacia adelante
        gpio_set_level(PIN_AIN1, 1);
        gpio_set_level(PIN_AIN2, 0);
        ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, speed);
    } else if (speed < 0) {
        // Hacia atrás (convertimos el negativo a positivo para el PWM)
        gpio_set_level(PIN_AIN1, 0);
        gpio_set_level(PIN_AIN2, 1);
        ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, abs(speed));
    } else {
        // Paro total
        gpio_set_level(PIN_AIN1, 0);
        gpio_set_level(PIN_AIN2, 0);
        ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, 0);
    }
    ledc_update_duty(LEDC_MODE, LEDC_CHANNEL);
}

static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data) {
    esp_mqtt_event_handle_t event = (esp_mqtt_event_handle_t) event_data;

    switch ((esp_mqtt_event_id_t)event_id) {
        case MQTT_EVENT_CONNECTED:
            esp_mqtt_client_subscribe(client, TOPIC_CMD, 0);
            esp_mqtt_client_publish(client, TOPIC_STAT, "{\"motor\":\"ready\"}", 0, 0, 1);
            break;

        case MQTT_EVENT_DATA: {
            char *raw_val = strndup(event->data, event->data_len);
            int speed = atoi(raw_val);
            
            if (speed >= -255 && speed <= 255) {
                ESP_LOGI(TAG, "Comando recibido: %d", speed);
                set_motor_speed(speed);
                
                char resp[64];
                snprintf(resp, sizeof(resp), "{\"current_speed\":%d}", speed);
                esp_mqtt_client_publish(client, TOPIC_STAT, resp, 0, 0, 0);
            }
            free(raw_val);
            break;
        }
        default: break;
    }
}

esp_err_t mqtt_app_start(const char *broker_uri) {
    init_motor_hardware();
    esp_mqtt_client_config_t cfg = { .broker.address.uri = broker_uri };
    client = esp_mqtt_client_init(&cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    return esp_mqtt_client_start(client);
}
```

#### Video of it working 
<!-- Ajuste para mostrar correctamente el video Shorts de YouTube como embed -->
<iframe width="560" height="315" src="https://www.youtube.com/embed/1R7rC5hZ4Ns" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
