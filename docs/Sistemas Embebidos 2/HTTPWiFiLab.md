# Session 4 - HTTP WiFI Lab
* Understand the use of the mutex

## 1) Activity Goal

[x] turn2 leds from a browser
[x] read a button and print it to the browser
[x] control the speed of a motor with a slider from the browser

- **Tools/Software** - Editors: VS Code, Python 3.12, ESP32-IDF

Wiring/Safety: ESP32-IDF, LED, BUTTON,MOTOR DC, H-BRIDGE RESISTANCE OF 1KOHM AND 220 OHMS

## 3) Procedure 

We start task by task, analyzing, how to adapt it to the previous one.

## 4) Data, Tests & Evidence
We understand from the evidence obtained that in some cases we have to change the priority of tasks for it to work properly, and the mutex part, could execute both tasks.

## 5) Analysis
Is necessary to asssign the priorities of the tasks in order for it to work properly, and to have a better organization of the tasks

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
// Control para el motor
#include "driver/ledc.h"
// HTTP server
#include "esp_http_server.h"

/* ===================== User config ===================== */
#define WIFI_SSID "iPhone de David"
#define WIFI_PASS "david888"

//Configuracion PWM PAR EL MOTOR
#define LEDC_TIMER              LEDC_TIMER_0
#define LEDC_MODE               LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL             LEDC_CHANNEL_0
#define LEDC_DUTY_RES           LEDC_TIMER_8_BIT // Set duty resolution to 13 bits
#define LEDC_FREQUENCY             (5000) // Set duty to 50%. ((2^13)-1)/2 = 4095/2 = 2047

#define LEDC_OUTPUT_IO          (4) // Define the output GPIO
/* LED pin */
#define LED_GPIO  3
#define LED_GPIO_2 2
#define BUTTON_GPIO 19

/* Reconnect policy */
#define MAX_RETRY 10

/* ===================== Globals ===================== */
static const char *TAG = "LAB_3";

// Wi-Fi event group and bit seen in Lab 2
static EventGroupHandle_t s_wifi_event_group;
#define WIFI_CONNECTED_BIT BIT0


static int s_retry = 0;// Retry count for Wi-Fi reconnects

static int s_led_state = 0;               // 0=OFF, 1=ON
static int s_led2_state = 0;               // 0=OFF, 1=ON
static int s_button_counter = 0;            // 0=not pressed, 1=pressed
static int s_button_last_state = 0;       // last data read


static httpd_handle_t s_server = NULL;    // HTTP server handle

/* ===================== LED helpers ===================== */
static void led_init(void)
{
        gpio_reset_pin(LED_GPIO);
        gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
        s_led_state = 0;
        gpio_set_level(LED_GPIO, s_led_state);

        gpio_reset_pin(LED_GPIO_2);
        gpio_set_direction(LED_GPIO_2, GPIO_MODE_OUTPUT);
        s_led2_state = 0;
        gpio_set_level(LED_GPIO_2, s_led2_state);
        


    ESP_LOGI(TAG, "LED initialized on GPIO %d (state=%d)", LED_GPIO, s_led_state);
    ESP_LOGI(TAG, "LED2 initialized on GPIO %d (state=%d)", LED_GPIO_2, s_led2_state);
}


static void button_init(void)
{
    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    // Agregamos pullup por seguridad si tu botón va a tierra
    gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);
    ESP_LOGI(TAG, "Button initialized on GPIO %d", BUTTON_GPIO);
}

//INICIALIZAMOS EL MOTOR
static void motor_set(void)
{
    // Configuración del timer para el canal LEDC
    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_MODE,
        .timer_num        = LEDC_TIMER,
        .duty_resolution  = LEDC_DUTY_RES,
        .freq_hz          = LEDC_FREQUENCY,
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ESP_ERROR_CHECK(ledc_timer_config(&ledc_timer));

    // Configuración del canal LEDC
    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_MODE,
        .channel        = LEDC_CHANNEL,
        .timer_sel      = LEDC_TIMER,
        .intr_type      = LEDC_INTR_DISABLE,
        .gpio_num       = LEDC_OUTPUT_IO,
        .duty           = 0, // Inicialmente apagado
        .hpoint         = 0
    };
    ESP_ERROR_CHECK(ledc_channel_config(&ledc_channel));

    ESP_LOGI(TAG, "Motor PWM initialized on GPIO %d", LEDC_OUTPUT_IO);
}

/*Function to set the LED from the state*/
static void led_set(int on)
{
    s_led_state = (on != 0);
    gpio_set_level(LED_GPIO, s_led_state);
    ESP_LOGI(TAG, "LED set to %d", s_led_state);
}

static void led2_set(int on)
{
    s_led2_state = (on != 0);
    gpio_set_level(LED_GPIO_2, s_led2_state);
    ESP_LOGI(TAG, "LED2 set to %d", s_led2_state);
}

static void button_task(void *pvParameters)
{
    while (1)
    {
        int current = gpio_get_level(BUTTON_GPIO);

        // Detectar flanco de bajada (botón presionado)
        if (s_button_last_state == 1 && current == 0)
        {
            s_button_counter++;
            ESP_LOGI(TAG, "Button pressed! Counter = %d", s_button_counter);
        }

        s_button_last_state = current;

        vTaskDelay(pdMS_TO_TICKS(50));  // debounce simple
    }
}
/* ===================== HTTP handlers ===================== */

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
    if (req->content_len <= 0 || req->content_len > 256) {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Invalid Content-Length");
        return ESP_FAIL;
    }
    char buf[257] = {0}; 
    int received = httpd_req_recv(req, buf, req->content_len);
    if (received <= 0) return ESP_FAIL;
    buf[received] = '\0';

    int state = -1;
    char *p = strstr(buf, "\"state\"");
    if (p) {
        p = strchr(p, ':');
        if (p) state = atoi(p + 1);
    }
    if (state != 0 && state != 1) {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "state must be 0 or 1");
        return ESP_FAIL;
    }
    led_set(state);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_sendstr(req, "{\"ok\":true}");
    return ESP_OK;
}

static esp_err_t api_led2_post(httpd_req_t *req)
{
    if (req->content_len <= 0 || req->content_len > 256) {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Invalid Content-Length");
        return ESP_FAIL;
    }
    char buf[257] = {0}; 
    int received = httpd_req_recv(req, buf, req->content_len);
    if (received <= 0) return ESP_FAIL;
    buf[received] = '\0';

    int state = -1;
    char *p = strstr(buf, "\"state\"");
    if (p) {
        p = strchr(p, ':');
        if (p) state = atoi(p + 1);
    }
    if (state != 0 && state != 1) {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "state must be 0 or 1");
        return ESP_FAIL;
    }
    led2_set(state);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_sendstr(req, "{\"ok\":true}");
    return ESP_OK;
}

static esp_err_t api_counter_get(httpd_req_t *req)
{
    char resp[32];
    snprintf(resp, sizeof(resp), "{\"counter\":%d}", s_button_counter);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, resp, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

static esp_err_t root_get_handler(httpd_req_t *req)
{
    // HTML INLINE CORREGIDO PARA MOSTRAR CONTADOR
    static const char *INDEX_HTML =
        "<!doctype html>\n"
        "<html>\n"
        "<head>\n"
        "  <meta charset='utf-8'>\n"
        "  <meta name='viewport' content='width=device-width, initial-scale=1'>\n"
        "  <title>ESP32-C6 Control</title>\n"
        "</head>\n"
        "<body>\n"
        "  <h2>ESP32-C6 LED Control</h2>\n"
        "  <button onclick='setLed(1)'>ON</button>\n"
        "  <button onclick='setLed(0)'>OFF</button>\n"
        "  <p id='st'>State: ?</p>\n"
        "  <hr>\n"
        "  <h3>LED 2</h3>\n"
        "  <button onclick='setLed2(1)'>ON</button>\n"
        "  <button onclick='setLed2(0)'>OFF</button>\n"
        "  <p id='st2'>State: ?</p>\n"
        "  <hr>\n"
        "  <h3>Contador de Botón</h3>\n"
        "  <p style='font-size: 2em;'>Pulsaciones: <span id='cnt'>0</span></p>\n"
        "\n"
        "  <script>\n"
        "    async function refresh(){\n"
        "      const r = await fetch('/api/led');\n"
        "      const j = await r.json();\n"
        "      document.getElementById('st').innerText = 'State: ' + j.state;\n"
        "    }\n"
        "    async function setLed(v){\n"
        "      await fetch('/api/led', {\n"
        "        method: 'POST',\n"
        "        headers: {'Content-Type':'application/json'},\n"
        "        body: JSON.stringify({state:v})\n"
        "      });\n"
        "      refresh();\n"
        "    }\n"
        "    async function refresh2(){\n"
        "      const r = await fetch('/api/led2');\n"
        "      const j = await r.json();\n"
        "      document.getElementById('st2').innerText = 'State: ' + j.state;\n"
        "    }\n"
        "    async function setLed2(v){\n"
        "      await fetch('/api/led2', {\n"
        "        method: 'POST',\n"
        "        headers: {'Content-Type':'application/json'},\n"
        "        body: JSON.stringify({state:v})\n"
        "      });\n"
        "      refresh2();\n"
        "    }\n"
        "    async function refreshCounter(){\n"
        "      const r = await fetch('/api/counter');\n"
        "      const j = await r.json();\n"
        "      document.getElementById('cnt').innerText = j.counter;\n"
        "    }\n"
        "    setInterval(refresh, 1000);\n"
        "    setInterval(refresh2, 1000);\n"
        "    setInterval(refreshCounter, 500);\n" // Refresca el contador cada medio segundo
        "    refresh(); refresh2(); refreshCounter();\n"
        "  </script>\n"
        "</body>\n"
        "</html>\n";

    httpd_resp_set_type(req, "text/html");
    httpd_resp_send(req, INDEX_HTML, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

static void http_server_start(void)
{
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(httpd_start(&s_server, &config));
    ESP_LOGI(TAG, "HTTP server started");

    httpd_uri_t root = { .uri="/", .method=HTTP_GET, .handler=root_get_handler };
    httpd_uri_t led_get = { .uri="/api/led", .method=HTTP_GET, .handler=api_led_get };
    httpd_uri_t led_post = { .uri="/api/led", .method=HTTP_POST, .handler=api_led_post };
    httpd_uri_t led2_get = { .uri="/api/led2", .method=HTTP_GET, .handler=api_led2_get };
    httpd_uri_t led2_post = { .uri="/api/led2", .method=HTTP_POST, .handler=api_led2_post };
    httpd_uri_t counter_get = { .uri="/api/counter", .method=HTTP_GET, .handler=api_counter_get };

    ESP_ERROR_CHECK(httpd_register_uri_handler(s_server, &counter_get));
    ESP_ERROR_CHECK(httpd_register_uri_handler(s_server, &root));
    ESP_ERROR_CHECK(httpd_register_uri_handler(s_server, &led_get));
    ESP_ERROR_CHECK(httpd_register_uri_handler(s_server, &led_post));
    ESP_ERROR_CHECK(httpd_register_uri_handler(s_server, &led2_get));
    ESP_ERROR_CHECK(httpd_register_uri_handler(s_server, &led2_post));

    ESP_LOGI(TAG, "Routes registered including /api/counter");
}

/* ===================== Wi-Fi STA connect + events ===================== */

static void wifi_event_handler(void *arg, esp_event_base_t event_base, int32_t event_id, void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        ESP_LOGI(TAG, "WIFI_EVENT_STA_START -> esp_wifi_connect()");
        ESP_ERROR_CHECK(esp_wifi_connect());
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry < MAX_RETRY) {
            s_retry++;
            ESP_LOGW(TAG, "Disconnected. Retrying (%d/%d)...", s_retry, MAX_RETRY);
            ESP_ERROR_CHECK(esp_wifi_connect());
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        ESP_LOGI(TAG, "Got IP: " IPSTR, IP2STR(&event->ip_info.ip));
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

    wifi_config_t wifi_config = {0};
    strncpy((char *)wifi_config.sta.ssid, WIFI_SSID, sizeof(wifi_config.sta.ssid));
    strncpy((char *)wifi_config.sta.password, WIFI_PASS, sizeof(wifi_config.sta.password));

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
}

/* ===================== app_main ===================== */

void app_main(void)
{
    ESP_LOGI(TAG, "Lab D start: Wi-Fi + HTTP + LED control.");
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ESP_ERROR_CHECK(nvs_flash_init());
    } else {
        ESP_ERROR_CHECK(ret);
    }

    wifi_init_sta();
    xEventGroupWaitBits(s_wifi_event_group, WIFI_CONNECTED_BIT, pdFALSE, pdTRUE, pdMS_TO_TICKS(30000));

    led_init();
    button_init();
    motor_set(); // Mantenemos tu configuración de motor
    http_server_start();

    xTaskCreate(button_task, "button_task", 2048, NULL, 5, NULL);
    ESP_LOGI(TAG, "Server Ready.");
}
```

## 7 files and media
* This is the video of it working.
<iframe width="560" height="315" src="https://www.youtube.com/embed/qte915losIw" frameborder="0" allowfullscreen></iframe>