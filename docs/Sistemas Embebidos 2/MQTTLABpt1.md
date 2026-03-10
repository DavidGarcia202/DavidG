# MQTT LAB pt1

## 1) Exercise Goals
[x]  Motor Control via MQTT

## 2) Materials & Setup
- **Tools/Software** - Editors: VS Code, Python 3.12, ESP32-C6

Wiring/Safety: ESP32-C6 , LED, BUTTON,MOTOR DC, H-BRIDGE RESISTANCE OF  220 OHMS

- **Hardware** - ESP32-C6, LED

## 3) Procedure 

## Lab 1
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
<iframe width="560" height="315" src="https://youtube.com/shorts/1R7rC5hZ4Ns?si=GTcgB341NPUGWonr" frameborder="0" allowfullscreen></iframe>
