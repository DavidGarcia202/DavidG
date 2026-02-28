# Session 4 - HTTP WiFI Lab
* Understand the use of the mutex

## 1) Activity Goal

[x] turn2 leds from a browser
[x] read a button and print it to the browser
[x] control the speed of a motor with a slider from the browser

- **Tools/Software** - Editors: VS Code, Python 3.12, ESP32-C6

Wiring/Safety: ESP32-C6 , LED, BUTTON,MOTOR DC, H-BRIDGE RESISTANCE OF 1KOHM AND 220 OHMS

## 3) Procedure 

We start by the led part, followed by the button part and finally we include the motor, joinig all the parts for a final code

## 4) Data, Tests & Evidence
We understand that we have to connect to a private wi-fi for it to workout, and following, we have to create each of the tasks independently, and the added to the http par.

## 5) Analysis
Only for the motor part, we must include another library, which is: #include "driver/ledc.h", because it is the one that the ESP-32 needs for it to control the motor, also, we need in the ht

## 6) Code
The final code done is the following one:
```bash
/*
 * LAB 3 — ESP32-C6 Wi-Fi + HTTP LED Control 
 *
 * Features:
 * - Wi-Fi STA connect + reconnect using events
 * - Wait for IP (WIFI_CONNECTED_BIT)
 * - Simple GPIO LED control (gpio_reset_pin + gpio_set_direction)
 * - HTTP server:
 * GET  /         -> HTML UI
 * GET  /api/led  -> JSON {"state":0|1}
 * POST /api/led  -> JSON {"state":0|1} sets LED, returns {"ok":true}
 *
 */

// String handling and standard libraries
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

// FreeRTOS and event groups
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"

// ESP-IDF Logging and error handling
#include "esp_log.h"
#include "esp_err.h"

// Wi-Fi and network
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "esp_wifi.h"

// GPIO control
#include "driver/gpio.h"

// HTTP server
#include "esp_http_server.h"

// Control para el motor
#include "driver/ledc.h"

/* ===================== User config ===================== */
#define WIFI_SSID "iPhone de David"
#define WIFI_PASS "david888"

// Configuracion PWM PARA EL MOTOR
#define LEDC_TIMER              LEDC_TIMER_0
#define LEDC_MODE               LEDC_LOW_SPEED_MODE 
#define LEDC_CHANNEL            LEDC_CHANNEL_0
#define LEDC_DUTY_RES           LEDC_TIMER_8_BIT // Resolucion de 8 bits (0-255)
#define LEDC_FREQUENCY          (5000)           // Frecuencia de 5kHz

// --- PINES DEL MOTOR  ---
#define LEDC_OUTPUT_IO          (0)  // PWM Speed en el pin 0
#define MOTOR_AIN1_GPIO         (7)  // Direccion AIN1 
#define MOTOR_AIN2_GPIO         (6)  // Direccion AIN2 

/* LED pins */
#define LED_GPIO                (11) // LED 1 
#define LED_GPIO_2              (12) // LED 2 
#define BUTTON_GPIO             (13) // Boton 

/* Reconnect policy */
#define MAX_RETRY 10

/* ===================== Globals ===================== */
static const char *TAG = "LAB_3_DAVID";

// Wi-Fi event group and bit seen in Lab 2
static EventGroupHandle_t s_wifi_event_group;
#define WIFI_CONNECTED_BIT BIT0

static int s_retry = 0;              // Retry count for Wi-Fi reconnects

static int s_led_state = 0;          // 0=OFF, 1=ON
static int s_led2_state = 0;         // 0=OFF, 1=ON
static int s_button_counter = 0;     // Contador de pulsaciones
static int s_button_last_state = 1;  // Ultimo estado leido (Pull-up)
static int s_motor_speed = 0;        // Velocidad actual (0-255)

static httpd_handle_t s_server = NULL; // HTTP server handle


/* ===================== LED helpers ===================== */
static void led_init(void)
{
    ESP_LOGI(TAG, "Configuring LED GPIOs...");
    
    // LED 1
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    // LED 2
    gpio_reset_pin(LED_GPIO_2);
    gpio_set_direction(LED_GPIO_2, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO_2, 0);
    
    ESP_LOGI(TAG, "LEDs initialized on pins %d and %d", LED_GPIO, LED_GPIO_2);
}

// CONFIGURACION DEL BOTON EN PULL-UP

static void button_init(void)
{
    ESP_LOGI(TAG, "Configuring Button GPIO...");
    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);
    ESP_LOGI(TAG, "Button initialized on pin %d", BUTTON_GPIO);
}

// MOTOR DC

static void motor_set(void)
{
    ESP_LOGI(TAG, "Configuring Motor Driver (PWM Pin 0, AIN1 Pin 7, AIN2 Pin 6)...");

    // Configuración de pines de dirección del puente H
    gpio_reset_pin(MOTOR_AIN1_GPIO);
    gpio_set_direction(MOTOR_AIN1_GPIO, GPIO_MODE_OUTPUT);
    gpio_reset_pin(MOTOR_AIN2_GPIO);
    gpio_set_direction(MOTOR_AIN2_GPIO, GPIO_MODE_OUTPUT);

    // Sentido de giro horario
    gpio_set_level(MOTOR_AIN1_GPIO, 1);
    gpio_set_level(MOTOR_AIN2_GPIO, 0);

    // Configuración del timer para el canal LEDC
    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_MODE,
        .timer_num        = LEDC_TIMER,
        .duty_resolution  = LEDC_DUTY_RES,
        .freq_hz          = LEDC_FREQUENCY,
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ESP_ERROR_CHECK(ledc_timer_config(&ledc_timer));

    // Configuración del canal LEDC para control de velocidad
    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_MODE,
        .channel        = LEDC_CHANNEL,
        .timer_sel      = LEDC_TIMER,
        .intr_type      = LEDC_INTR_DISABLE,
        .gpio_num       = LEDC_OUTPUT_IO,
        .duty           = 0, 
        .hpoint         = 0
    };
    ESP_ERROR_CHECK(ledc_channel_config(&ledc_channel));

    ESP_LOGI(TAG, "Motor hardware ready.");
}

// DUTY CYCLE 
static void motor_update_speed(int speed)
{
    if (speed < 0) speed = 0;
    if (speed > 255) speed = 255;
    
    s_motor_speed = speed;
    ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, s_motor_speed);
    ledc_update_duty(LEDC_MODE, LEDC_CHANNEL);
    ESP_LOGI(TAG, "New speed set: %d", s_motor_speed);
}

// Function to set the LED state
static void led_set(int on)
{
    s_led_state = (on != 0);
    gpio_set_level(LED_GPIO, s_led_state);
    ESP_LOGI(TAG, "LED1 set to %d", s_led_state);
}

// Function to set the LED2 state
static void led2_set(int on)
{
    s_led2_state = (on != 0);
    gpio_set_level(LED_GPIO_2, s_led2_state);
    ESP_LOGI(TAG, "LED2 set to %d", s_led2_state);
}

// FreeRTOS task to monitor the button

static void button_task(void *pvParameters)
{
    ESP_LOGI(TAG, "Button monitor task started.");
    while (1)
    {
        int current = gpio_get_level(BUTTON_GPIO);

        // Detectar flanco de bajada (botón presionado con pull-up)
        if (s_button_last_state == 1 && current == 0)
        {
            s_button_counter++;
            ESP_LOGI(TAG, "Button EVENT: Counter incremented to %d", s_button_counter);
        }

        s_button_last_state = current;
        vTaskDelay(pdMS_TO_TICKS(50));  // Simple debounce delay
    }
}

/* ===================== HTTP Handlers ===================== */
// --- MOTOR HANDLERS ---
static esp_err_t api_motor_get(httpd_req_t *req)
{
    char resp[64];
    snprintf(resp, sizeof(resp), "{\"speed\":%d}", s_motor_speed);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, resp, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

static esp_err_t api_motor_post(httpd_req_t *req)
{
    char buf[128];
    int received = httpd_req_recv(req, buf, req->content_len);
    if (received <= 0) return ESP_FAIL;
    buf[received] = '\0';

    int val = 0;
    char *p = strstr(buf, "\"speed\"");
    if (p) {
        p = strchr(p, ':');
        if (p) val = atoi(p + 1);
    }
    motor_update_speed(val);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_sendstr(req, "{\"ok\":true}");
    return ESP_OK;
}

// --- LED HANDLERS ---
static esp_err_t api_led_get(httpd_req_t *req)
{
    char resp[32];
    snprintf(resp, sizeof(resp), "{\"state\":%d}", s_led_state);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, resp, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

static esp_err_t api_led2_get(httpd_req_t *req)
{
    char resp[32];
    snprintf(resp, sizeof(resp), "{\"state\":%d}", s_led2_state);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, resp, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

static esp_err_t api_led_post(httpd_req_t *req)
{
    char buf[257] = {0}; 
    int received = httpd_req_recv(req, buf, req->content_len);
    if (received <= 0) return ESP_FAIL;
    buf[received] = '\0';

    int state = -1;
    char *p = strstr(buf, "\"state\"");
    if (p) { p = strchr(p, ':'); if (p) state = atoi(p + 1); }
    led_set(state);
    httpd_resp_sendstr(req, "{\"ok\":true}");
    return ESP_OK;
}

static esp_err_t api_led2_post(httpd_req_t *req)
{
    char buf[257] = {0}; 
    int received = httpd_req_recv(req, buf, req->content_len);
    if (received <= 0) return ESP_FAIL;
    buf[received] = '\0';

    int state = -1;
    char *p = strstr(buf, "\"state\"");
    if (p) { p = strchr(p, ':'); if (p) state = atoi(p + 1); }
    led2_set(state);
    httpd_resp_sendstr(req, "{\"ok\":true}");
    return ESP_OK;
}

// --- COUNTER HANDLER ---
static esp_err_t api_counter_get(httpd_req_t *req)
{
    char resp[32];
    snprintf(resp, sizeof(resp), "{\"counter\":%d}", s_button_counter);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, resp, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

// --- MAIN HTML HANDLER ---
static esp_err_t root_get_handler(httpd_req_t *req)
{
    static const char *INDEX_HTML =
        "<!doctype html><html><head><meta charset='utf-8'>"
        "<meta name='viewport' content='width=device-width, initial-scale=1'>"
        "<style>body{font-family:sans-serif; text-align:center; background:#f4f4f9; padding:20px;}"
        ".box{background:white; padding:20px; border-radius:10px; box-shadow:0 2px 10px rgba(0,0,0,0.1); margin-bottom:20px;}"
        "button{padding:10px 20px; margin:5px; border-radius:5px; border:none; background:#007bff; color:white; cursor:pointer;}"
        "button:active{background:#0056b3;} input[type=range]{width:80%;}</style>"
        "<title>ESP32-C6 Lab David</title></head><body>"
        "<h2>Sistema de Control Web</h2>"
        
        "<div class='box'><h3>LED 1</h3>"
        "<button onclick='setLed(1)'>ENCENDER</button><button onclick='setLed(0)'>APAGAR</button>"
        "<p id='st'>Estado: ?</p></div>"

        "<div class='box'><h3>LED 2</h3>"
        "<button onclick='setLed2(1)'>ENCENDER</button><button onclick='setLed2(0)'>APAGAR</button>"
        "<p id='st2'>Estado: ?</p></div>"

        "<div class='box'><h3>Control Motor </h3>"
        "<input type='range' min='0' max='255' value='0' oninput='setMotor(this.value)'>"
        "<p>Velocidad Actual: <b id='mv'>0</b></p></div>"

        "<div class='box'><h3>Contador de Boton</h3>"
        "<p style='font-size:2em;' id='cnt'>0</p></div>"

        "<script>\n"
        "  async function refresh(){\n"
        "    const r1 = await fetch('/api/led'); const j1 = await r1.json();\n"
        "    document.getElementById('st').innerText = 'Estado: ' + j1.state;\n"
        "    const r2 = await fetch('/api/led2'); const j2 = await r2.json();\n"
        "    document.getElementById('st2').innerText = 'Estado: ' + j2.state;\n"
        "    const r3 = await fetch('/api/counter'); const j3 = await r3.json();\n"
        "    document.getElementById('cnt').innerText = j3.counter;\n"
        "    const r4 = await fetch('/api/motor'); const j4 = await r4.json();\n"
        "    document.getElementById('mv').innerText = j4.speed;\n"
        "  }\n"
        "  async function setLed(v){ await fetch('/api/led', {method:'POST', body:JSON.stringify({state:v})}); refresh(); }\n"
        "  async function setLed2(v){ await fetch('/api/led2', {method:'POST', body:JSON.stringify({state:v})}); refresh(); }\n"
        "  async function setMotor(v){ document.getElementById('mv').innerText = v; await fetch('/api/motor', {method:'POST', body:JSON.stringify({speed:parseInt(v)})}); }\n"
        "  setInterval(refresh, 800);\n"
        "  refresh();\n"
        "</script></body></html>";

    httpd_resp_set_type(req, "text/html");
    httpd_resp_send(req, INDEX_HTML, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

/* ===================== HTTP Server Start + Route Registration ===================== */
static void http_server_start(void)
{
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    config.stack_size = 8192; // Aumentar stack para mayor estabilidad

    ESP_LOGI(TAG, "Starting HTTP Server...");
    if (httpd_start(&s_server, &config) == ESP_OK) {
        
        httpd_uri_t root = { .uri="/", .method=HTTP_GET, .handler=root_get_handler };
        httpd_uri_t led_get = { .uri="/api/led", .method=HTTP_GET, .handler=api_led_get };
        httpd_uri_t led_post = { .uri="/api/led", .method=HTTP_POST, .handler=api_led_post };
        httpd_uri_t led2_get = { .uri="/api/led2", .method=HTTP_GET, .handler=api_led2_get };
        httpd_uri_t led2_post = { .uri="/api/led2", .method=HTTP_POST, .handler=api_led2_post };
        httpd_uri_t motor_get = { .uri="/api/motor", .method=HTTP_GET, .handler=api_motor_get };
        httpd_uri_t motor_post = { .uri="/api/motor", .method=HTTP_POST, .handler=api_motor_post };
        httpd_uri_t counter_get = { .uri="/api/counter", .method=HTTP_GET, .handler=api_counter_get };

        httpd_register_uri_handler(s_server, &root);
        httpd_register_uri_handler(s_server, &led_get);
        httpd_register_uri_handler(s_server, &led_post);
        httpd_register_uri_handler(s_server, &led2_get);
        httpd_register_uri_handler(s_server, &led2_post);
        httpd_register_uri_handler(s_server, &motor_get);
        httpd_register_uri_handler(s_server, &motor_post);
        httpd_register_uri_handler(s_server, &counter_get);
        
        ESP_LOGI(TAG, "All HTTP routes registered successfully.");
    }
}

/* ===================== Wi-Fi STA connect + events ===================== */

static void wifi_event_handler(void *arg, esp_event_base_t event_base, int32_t event_id, void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        ESP_LOGI(TAG, "Wi-Fi connecting...");
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry < MAX_RETRY) {
            s_retry++;
            ESP_LOGW(TAG, "Retry connection to AP (%d/%d)", s_retry, MAX_RETRY);
            esp_wifi_connect();
        } else {
            ESP_LOGE(TAG, "Failed to connect to Wi-Fi after maximum retries.");
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        ESP_LOGI(TAG, "CONNECTED! IP address: " IPSTR, IP2STR(&event->ip_info.ip));
        s_retry = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

static void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();
    
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
        },
    };

    ESP_LOGI(TAG, "Setting Wi-Fi configuration SSID: %s", (char *)wifi_config.sta.ssid);
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
}

/* ===================== app_main ===================== */

void app_main(void)
{
    ESP_LOGI(TAG, "[APP_MAIN] Starting Lab 3 Project...");

    // Initialize NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // Start Wi-Fi
    wifi_init_sta();

    // Wait for connection
    ESP_LOGI(TAG, "Waiting for Wi-Fi connection bit...");
    xEventGroupWaitBits(s_wifi_event_group, WIFI_CONNECTED_BIT, pdFALSE, pdTRUE, portMAX_DELAY);

    // Initialize Peripherals
    led_init();
    button_init();
    motor_set();

    // Start Server
    http_server_start();

    // Start Tasks
    xTaskCreate(button_task, "button_task", 4096, NULL, 5, NULL);

    ESP_LOGI(TAG, "Initialization complete");
}
```

## 7 files and media
* This is the video of it working.

<iframe width="560" height="315" src="https://www.youtube.com/embed/mEHZSc8GDS0?si=034hMeJtgWENcsfP" frameborder="0" allowfullscreen></iframe>

# Lab1 Questions:

## Screenshot of Serial Terminal

![LAB1](LAB1_2602.jpg)

## Answers

### 1. What is the SSID of the strongest AP detected in your scan?

**R =** IBERO, IBERO_IOT, IBERO CORTESIA, IBERO_INVITADOS  

---

### 2. What is the RSSI value of that AP?

**R =** -44  

---

### 3. What authentication mode does that AP use?

**R =** WPA2-ENT

---

### 4. What channel is that AP operating on?

**R =** 11  

---

### 5. Which channel had the most APs detected in your scan?

**R =** 1  

---


# Lab 2 Questions: 

### 1. What event indicates “Wi-Fi driver started and is ready to connect”? (event name)

*(Already answered above)*

---

### 2. What event indicates “network stack has an IP and the device is online”? (event name)

**R =** IP_EVENT_STA_GOT_IP  

---

### 3. What is the assigned IP address?

**R =** 172.20.10.12  

---

### 4. How many retries occurred before success or failure?

**R =** Retries = 0  

---

### 5. In your logs, which happened first: WIFI_EVENT_STA_START or IP_EVENT_STA_GOT_IP? Why?

**R =** WIFI_EVENT_STA_START.  
The device must start the Wi-Fi driver and establish a connection before it can obtain an IP address.